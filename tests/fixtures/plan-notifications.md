# Plan: User Notification Center

## Goal

Add an in-app notification center so users see system events (e.g., "your order shipped", "invoice paid") without polling or refreshing the page. The notification center is a header element that shows unread counts and opens a drop-down list on click.

## Scope

- One new database table for notifications
- One new backend endpoint to list a user's notifications
- One new backend endpoint to mark a notification as read
- A frontend notification bell component in the global header
- A frontend drop-down list component showing the latest 20 notifications
- Real-time updates are OUT of scope for this iteration — users see new notifications on next page load or on manual refresh of the drop-down

## Requirements

### Functional

1. Logged-in users see a bell icon in the site header at all times
2. The bell shows an unread-count badge when the user has any unread notifications (1–99, or "99+" for higher)
3. Clicking the bell opens a drop-down listing the 20 most recent notifications (read and unread mixed, newest first)
4. Each notification row shows: a title, a short body, and a relative timestamp ("5m ago")
5. Unread rows are visually distinct from read rows
6. Clicking a notification row marks it as read and navigates the user to the linked resource (if a URL is set)
7. A "Mark all as read" button is shown above the list when there is at least one unread item
8. The drop-down can be dismissed by clicking outside it or pressing Escape
9. The feature works on mobile widths (drop-down collapses to full-screen sheet below 640px)

### Non-Functional

1. All endpoints require authentication (unauthenticated requests return 401)
2. A user can only see and mark their own notifications (authorization enforced at the query layer)
3. The list endpoint paginates at 20 items and returns a total unread count in the response metadata
4. The bell component is keyboard accessible — Tab to focus, Enter/Space to open, Escape to close, Arrow keys to move through rows
5. Unread count updates announce via `aria-live="polite"` for screen reader users
6. Response time budget: list endpoint returns in under 150ms at p95 for users with up to 10k notifications

## Data Model

```
notifications
  id              uuid primary key
  user_id         uuid not null references users(id)
  title           text not null
  body            text not null
  link_url        text nullable
  read_at         timestamptz nullable
  created_at      timestamptz not null default now()

  index on (user_id, created_at desc)
  index on (user_id) where read_at is null
```

## API Contract Sketch

- `GET /api/v1/notifications` — returns the latest 20 notifications for the authenticated user, plus `meta.unread_count`
- `POST /api/v1/notifications/:id/read` — marks the notification as read if it belongs to the authenticated user, returns 204
- `POST /api/v1/notifications/read-all` — marks all of the user's notifications as read, returns 204

## Acceptance Criteria

- [ ] `notifications` table exists with the columns and indexes above
- [ ] All three endpoints pass integration tests covering happy path, auth failures, and cross-user isolation (user A cannot mark user B's notification as read)
- [ ] The bell component renders, shows the correct unread count, and meets WCAG 2.1 AA keyboard/screen-reader expectations
- [ ] The drop-down list renders loading, empty, and populated states
- [ ] Clicking a row marks it read AND navigates to `link_url` if present
- [ ] "Mark all as read" endpoint is wired to a button that becomes disabled when unread count is 0
- [ ] E2E test covers the critical journey: log in, see unread badge, click bell, click a notification, land on target page, badge decrements
- [ ] Test coverage for new code meets 80% minimum

## Out of Scope

- Push notifications / email / SMS delivery channels
- Notification preferences UI
- Admin tools for creating notifications (seed data is enough for this plan)
- Real-time WebSocket or SSE updates
- Internationalization of notification text
