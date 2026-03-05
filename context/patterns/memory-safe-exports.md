# Memory-Safe Data Processing in Background Jobs

When background jobs process medium-to-large datasets (CSV exports, XML imports, chat dumps), the entire payload must never live on the Ruby heap at once. This document codifies the patterns used across all export and import jobs.

## Core Principles

1. **Stream to disk, not RAM** — Write rows/chunks to a `Tempfile` instead of accumulating in a `String` (e.g., `CSV.generate` or `response.body`).
2. **Use the persistent volume** — On Fly.io workers, `/tmp` is an in-memory tmpfs. Write tempfiles to `/data/tmp` (ext4 volume) so large files don't count against worker memory.
3. **Upload to S3 via ActiveStorage** — Attach the tempfile to an `Export` record, then email a signed download link instead of a file attachment. This avoids loading the file into memory again at delivery time.
4. **Always clean up** — Use `ensure` blocks to call `tempfile.close!` so disk space is reclaimed even on failure.

---

## Pattern 1: CsvExportable Concern (Tempfile Helper)

All models and services that produce CSV files include the `CsvExportable` concern to get a tempfile pointed at the right directory.

```ruby
# app/models/concerns/csv_exportable.rb
module CsvExportable
  extend ActiveSupport::Concern

  def csv_tempfile(prefix)
    tmpdir = Dir.exist?('/data') ? '/data/tmp' : Dir.tmpdir
    FileUtils.mkdir_p(tmpdir)
    Tempfile.new([prefix, '.csv'], tmpdir)
  end
end
```

**Key details:**
- Falls back to `Dir.tmpdir` for local development (no `/data` volume).
- Caller is responsible for `tempfile.close!` in an `ensure` block.

---

## Pattern 2: Streaming CSV Generation (Model Method)

Replace `CSV.generate` (builds entire CSV in a String) with `CSV.new(tempfile)` (writes each row directly to disk).

### Before (memory-heavy)
```ruby
def to_csv
  CSV.generate(headers: true) do |csv|
    csv << headers
    records.each { |r| csv << row_for(r) }
  end
  # Returns a String containing the entire CSV
end
```

### After (memory-safe)
```ruby
include CsvExportable

def to_csv_file
  tempfile = csv_tempfile('my_export')
  csv = CSV.new(tempfile)
  csv << headers

  records.find_each do |record|   # find_each batches in groups of 1000
    csv << row_for(record)
  end

  tempfile.rewind
  tempfile
rescue StandardError => e
  tempfile&.close!
  Rails.logger.error("MyModel#to_csv_file error: #{e.message}")
  raise
end
```

**Key details:**
- Name the method `to_csv_file` (returns a Tempfile) to distinguish from `to_csv` (returns a String).
- Use `find_each` instead of `each` for ActiveRecord collections — it loads records in batches.
- Call `tempfile.rewind` before returning so the caller can read from the beginning.
- The `rescue` block cleans up the tempfile before re-raising.

---

## Pattern 3: Export Model + S3 Upload + Download Link

Instead of emailing CSV data as an attachment (loads entire file into memory for the mailer), create an `Export` record, attach the file to S3 via ActiveStorage, and email a signed download URL.

### Export Model
```ruby
# app/models/export.rb
class Export < ApplicationRecord
  belongs_to :staff, class_name: 'User'
  belongs_to :organization
  has_one_attached :file

  validates :export_type, presence: true
  validates :filename, presence: true
  validates :status, presence: true, inclusion: { in: %w[pending completed failed] }

  def completed!
    update!(status: 'completed')
  end

  def failed!
    update!(status: 'failed')
  end

  def download_url
    Rails.application.routes.url_helpers.export_download_url(
      signed_id: signed_id(expires_in: 7.days),
      host: ENV.fetch('DOMAIN'),
      protocol: 'https'
    )
  end
end
```

### Download Controller (signed URL authorization)
```ruby
# app/controllers/export_downloads_controller.rb
class ExportDownloadsController < ApplicationController
  def show
    export = Export.find_signed!(params[:signed_id])
    redirect_to rails_blob_url(export.file, disposition: :attachment), allow_other_host: true
  rescue ActiveSupport::MessageVerifier::InvalidSignature
    render plain: 'This download link has expired or is invalid.', status: :unauthorized
  end
end
```

Security is enforced via the signed URL (7-day expiry), not Pundit policies.

---

## Pattern 4: Standard Export Job Structure

Every export job follows the same structure: create Export → generate tempfile → attach to S3 → mark complete → email link → clean up.

```ruby
class MyExportJob < ApplicationJob
  def perform(record_id, staff_id)
    staff = Staff.find(staff_id)
    record = staff.organization.my_records.find(record_id)

    filename = record.csv_filename
    export = Export.create!(
      export_type: 'my_export',
      filename: filename,
      staff: staff,
      organization: staff.organization
    )

    tempfile = record.to_csv_file  # Pattern 2

    export.file.attach(
      io: tempfile,
      filename: filename,
      content_type: 'text/csv'
    )

    export.completed!

    ExportMailer.with(export: export).export_download.deliver_now
  rescue StandardError
    export&.failed! if export&.persisted?
    raise
  ensure
    tempfile&.close!
  end
end
```

**Key details:**
- `deliver_now` (not `deliver_later`) — the job itself is already async; double-queuing adds unnecessary memory risk.
- The `rescue` marks the Export as failed so the UI/admin can see it.
- The `ensure` block always cleans up the tempfile, even on success.

---

## Pattern 5: Streaming HTTP Responses to Disk (XML Imports)

For large HTTP responses (XML job feeds, 20-50 MB), stream the response body to disk chunk-by-chunk instead of buffering in memory.

```ruby
# app/clients/xml/client.rb
def get_to_file(url:)
  tmpdir = Dir.exist?('/data') ? '/data/tmp' : Dir.tmpdir
  FileUtils.mkdir_p(tmpdir)
  tempfile = Tempfile.new(['xml_feed', '.xml'], tmpdir)
  tempfile.binmode

  uri = URI(url)
  Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == 'https') do |http|
    request = Net::HTTP::Get.new(uri)
    http.request(request) do |response|
      raise "HTTP #{response.code} fetching #{url}" unless response.is_a?(Net::HTTPSuccess)
      response.read_body { |chunk| tempfile.write(chunk) }
    end
  end

  tempfile.rewind
  tempfile
rescue
  tempfile&.close!
  raise
end
```

**Key details:**
- Use `Net::HTTP` with a block form of `http.request` to enable chunked reading.
- Call `tempfile.binmode` for binary-safe writes.
- Validate HTTP status before reading the body.
- Clean up the tempfile on failure to prevent orphaned files.

---

## Pattern 6: Controller CSV Export via Tempfile

For synchronous (non-job) CSV downloads in controllers, use `send_file` with a tempfile instead of `send_data` with a string.

```ruby
class MyController < ApplicationController
  after_action :cleanup_export_tempfile, only: :export

  def export
    result = build_csv_export
    @export_tempfile = result[:tempfile]
    send_file @export_tempfile.path, filename: result[:filename], type: 'text/csv', disposition: 'attachment'
  end

  private

  def cleanup_export_tempfile
    @export_tempfile&.close!
  end

  def build_csv_export
    tempfile = Tempfile.new(['report', '.csv'])
    csv = CSV.new(tempfile)
    csv << headers
    data.each { |row| csv << row }
    tempfile.flush
    tempfile.rewind
    { tempfile: tempfile, filename: "report-#{Date.current}.csv" }
  end
end
```

**Key details:**
- `send_file` streams from disk; `send_data` loads into memory.
- `after_action` ensures cleanup happens after the response is sent.
- Call `tempfile.flush` before `rewind` to ensure all data is written.

---

## Pattern 7: Caching Expensive Queries in Batch Operations

When a service is called N times inside a loop and each call runs the same expensive query, hoist the query and pass the result in.

### Before (N+1 query on every job in an import loop)
```ruby
saved.each do |resource_link|
  ShareOrganizationResource.new(resource_link: resource_link, ...).run!
end

# Inside ShareOrganizationResource:
def share_with_network
  all_org_ids = Organization.all.pluck(:id)  # Runs every time!
end
```

### After (query once, pass result)
```ruby
network_org_ids = Organization.pluck(:id)  # Once

saved.each do |resource_link|
  ShareOrganizationResource.new(
    resource_link: resource_link,
    network_org_ids: network_org_ids,  # Reuse cached result
    ...
  ).run!
end

# Inside ShareOrganizationResource:
def share_with_network
  all_org_ids = @network_org_ids || Organization.all.pluck(:id)
end
```

Use memoization (`@network_org_ids ||=`) at the import-loop level so the array is allocated once per import run.

---

## Pattern 8: Push Filtering to SQL

Replace Ruby-side filtering with SQL to avoid loading unnecessary records into memory.

### Before (loads all records, filters in Ruby)
```ruby
group = candidates.where.not(date_of_birth: nil).select do |c|
  dob = c.date_of_birth
  range.include?(((today - dob) / 365.25).floor)
end
total = group.size
count = group.count { |c| c.barriers&.include?(barrier) }
```

### After (SQL does the filtering)
```ruby
group = candidates.where.not(date_of_birth: nil)
  .where("DATE_PART('year', AGE(date_of_birth)) BETWEEN ? AND ?", range.begin, range.end)
total = group.count
count = group.where("barriers @> ?", [barrier].to_json).count
```

This keeps only the counts in Ruby — no record instantiation at all.

---

## Pattern 9: Eager Loading to Eliminate N+1 Queries

When iterating over records to build CSV rows, eager-load all associations accessed in the loop.

```ruby
def candidates(user:)
  CandidatesQuery.new(
    user: user,
    includes: [:profile, :assigned_users, :groups, :users_organizations, { fact_values: :fact_option }],
    ...
  )
end
```

Without eager loading, each `candidate.profile`, `candidate.groups`, etc. fires a separate SQL query per row.

---

## Checklist for New Exports

When adding a new CSV export or data processing job:

- [ ] Use `CsvExportable` concern and `csv_tempfile` helper
- [ ] Write CSV rows with `CSV.new(tempfile)`, not `CSV.generate`
- [ ] Use `find_each` for ActiveRecord iteration (batched loading)
- [ ] Eager-load all associations used in the row builder
- [ ] Create an `Export` record with status tracking
- [ ] Attach file to S3 via ActiveStorage (`export.file.attach`)
- [ ] Email a signed download link, not a file attachment
- [ ] Use `deliver_now` (job is already async)
- [ ] `rescue` → mark export as failed
- [ ] `ensure` → `tempfile.close!`
- [ ] For HTTP responses > 1 MB, stream to disk (Pattern 5)
- [ ] Push filtering/aggregation to SQL where possible
