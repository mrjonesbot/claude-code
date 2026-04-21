# Development Workflow & Tooling

## Browser Tools for Visual Checks

Two browser tools are available depending on the terminal environment. Use whichever is available — both support navigation, inspection, interaction, and snapshots.

### Detecting Your Environment

- **cmux terminal**: The `CMUX` binary exists at `/Applications/cmux.app/Contents/Resources/bin/cmux`. Prefer cmux browser — it renders in a split pane alongside the terminal.
- **tmux / other terminals**: Use the Playwright MCP tools (`mcp__playwright__browser_*`). These run a headless Chromium browser.

---

### Playwright MCP (use in tmux / non-cmux terminals)

Runs a headless Chromium browser via MCP tools. All tools are prefixed with `mcp__playwright__browser_`.

**Key tools:**
| Action | Tool |
|---|---|
| Navigate to URL | `mcp__playwright__browser_navigate` |
| Go back | `mcp__playwright__browser_navigate_back` |
| Go forward | `mcp__playwright__browser_navigate_forward` |
| Accessibility snapshot | `mcp__playwright__browser_snapshot` |
| Take screenshot | `mcp__playwright__browser_take_screenshot` |
| Click element | `mcp__playwright__browser_click` |
| Type text | `mcp__playwright__browser_type` |
| Fill form fields | `mcp__playwright__browser_fill_form` |
| Select dropdown option | `mcp__playwright__browser_select_option` |
| Press key | `mcp__playwright__browser_press_key` |
| Evaluate JavaScript | `mcp__playwright__browser_evaluate` |
| Console messages | `mcp__playwright__browser_console_messages` |
| Wait for text/element | `mcp__playwright__browser_wait_for` |
| Resize viewport | `mcp__playwright__browser_resize` |
| Close browser | `mcp__playwright__browser_close` |

**Notes:**
- Playwright tools interact via element `ref` values from `browser_snapshot`, not CSS selectors
- Use `browser_snapshot` (accessibility tree) for understanding page state and getting element refs
- Use `browser_take_screenshot` when you need a visual capture
- If you get an error about the browser not being installed, call `mcp__playwright__browser_install` first
- **CRITICAL — screenshot cleanup**: Never pass a bare `filename` to `browser_take_screenshot` (e.g., `"dashboard-check.png"`). This dumps files in the project root. Either omit `filename` entirely (auto-saves to `.playwright-mcp/`) or prefix with `.playwright-mcp/` (e.g., `".playwright-mcp/dashboard-check.png"`). A Stop hook auto-cleans `.playwright-mcp/` at session end.

---

## Quick Visual Check

IMMEDIATELY after implementing any front-end change:
1. **Identify what changed** - Review the modified components/pages
2. **Navigate to affected pages** - Open each changed view in the browser (cmux: `$CMUX browser open <url>`, Playwright: `browser_navigate`)
3. **Verify design compliance** - Compare against `/context/design-principles.md` and `/context/style-guide.md`
4. **Validate feature implementation** - Ensure the change fulfills the user's specific request
5. **Check acceptance criteria** - Review any provided context files or requirements
6. **Capture evidence** - Take a snapshot to inspect page state (cmux: `$CMUX browser snapshot`, Playwright: `browser_snapshot`)
7. **Check for errors** - Check console messages or dev logs

This verification ensures changes meet design standards and user requirements.

## Comprehensive Design Review

Invoke the `@agent-design-review` subagent for thorough design validation when:
- Completing significant UI/UX features
- Before finalizing PRs with visual changes
