# Turbo Stream Broadcasts from Background Jobs

Pattern for broadcasting Turbo Stream updates from background jobs (SolidQueue, etc.) to update the UI in real time.

## Key Rules

1. **Always wrap broadcasts in `rescue StandardError`** — broadcast failures must never crash or retry the job
2. **Use `broadcast_update_to`** for updating a container's contents (keeps the wrapper element)
3. **Use `broadcast_replace_to`** for replacing an entire element including its wrapper (partial must render a new element with the same DOM ID)
4. **Stream subscriptions** — the target view MUST have a matching `turbo_stream_from` tag or the broadcast has no listeners

## Reference Implementation (Nestingbird)

```ruby
# Broadcasting a "in-progress" state
def broadcast_importing(stream)
  Turbo::StreamsChannel.broadcast_update_to(
    stream,
    target: "initial_import_container",
    partial: "transactions/importing"
  )
rescue StandardError => e
  Rails.logger.error("Failed to broadcast importing state: #{e.message}")
end

# Broadcasting completion (with page redirect via Turbo.visit)
def broadcast_complete(stream)
  Turbo::StreamsChannel.broadcast_update_to(
    stream,
    target: "initial_import_container",
    html: "<script>Turbo.visit('#{Rails.application.routes.url_helpers.transactions_path}')</script>"
  )
rescue StandardError => e
  Rails.logger.error("Failed to broadcast import completion: #{e.message}")
end
```

## Stream Naming

```ruby
# View subscribes (controller sets @current_account):
turbo_stream_from @current_account, :integration_connections

# Job broadcasts to same stream:
Turbo::StreamsChannel.broadcast_replace_to(
  integration.account, :integration_connections,
  target: dom_id(integration, :card),
  partial: "integration_connections/integration_card",
  locals: { integration: integration, provider: provider }
)
```

Stream arguments must match exactly between `turbo_stream_from` and `broadcast_*_to`.

## Cable Adapter Requirements

- **Development**: Use `solid_cable` adapter (not `async`) so broadcasts work across processes (e.g., SolidQueue worker → Puma)
- **Production**: Use `solid_cable` with `polling_interval: 0.1.seconds`
- The `async` adapter only works within a single process — broadcasts from a separate job worker will never reach the web process
