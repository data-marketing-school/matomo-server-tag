# Matomo Analytics tag for server-side Google Tag Manager

A tag template for server-side Google Tag Manager (sGTM). It sends hits to Matomo's tracking API (`/matomo.php`) from your tagging server. It works with two kinds of client:

- **GA4 client**: turns GA4 events (page views, events, e-commerce) into Matomo hits.
- **Matomo client**: forwards hits that the Matomo JavaScript tracker sent to your tagging server, for example through the [Matomo client template](https://github.com/data-marketing-school/matomo-client).

Full guide: [Matomo server-side tracking](https://data-marketing-school.com/en/blog/google-tag-manager/matomo-server-side/).

## How hits are sent

Every hit is sent as a `POST` to `<instance URL>/matomo.php` through Matomo's [bulk tracking API](https://developer.matomo.org/api-reference/tracking-api#bulk-tracking), with a JSON body:

```json
{
  "requests": [{ "idsite": "1", "rec": 1, "action_name": "Home", "url": "https://example.com/", "_id": "…" }],
  "token_auth": "…"
}
```

- Each hit is a decoded parameter object, so GTM Server preview shows readable values on the outgoing request.
- `token_auth` sits at the top level of the body and is never put in the URL, so it stays out of access logs.
- The tag fails if Matomo returns a non-2xx status, or a 2xx response that reports rejected hits (`"invalid"` greater than 0).

## Configuration

| Field | Description |
|---|---|
| Client Type | `GA4 Client` or `Matomo Client`: the client whose events this tag sends. |
| Matomo Instance URL | Base URL of your Matomo instance, e.g. `https://example.matomo.cloud`. `/matomo.php` is added automatically. |
| Auth Token | A Matomo `token_auth` with write access to the site. Matomo needs it to accept the visitor IP (`cip`) sent from the server. |
| Site ID | The Matomo site ID. With the Matomo client it is optional and replaces the `idsite` from the incoming hit. |

### GA4 client

**Event Type**

- **Page View**: a page view built from `page_location`, `page_title` and `page_referrer`.
- **Actions**:
  - **Event**: category, action, name and value.
  - **Goal**: goal ID and revenue.
  - **Link**: an outlink or a download.
  - **Site Search**: keyword (default: `search_term`), category and number of results.
- **E-commerce Tracking**:
  - **Category View** and **Product View**: use the first item of `items` by default.
  - **Cart Update**: sends the full cart. The tag keeps the cart in a server-side cookie (`mtm_cart_155c56`), updated from GA4 `add_to_cart` and `remove_from_cart` events.
  - **Ecommerce Order**: needs an order ID (default: `transaction_id`), otherwise the tag is not sent. Items, value, tax, shipping and discount default to the GA4 event data; subtotal is only sent when set. The cart cookie is cleared after the order.
- **Content Tracking**: a content impression or interaction.
- **Heartbeat**: a ping that measures time spent on the page.

**Custom Dimensions**: a table of dimension IDs and values, sent as `dimension<ID>`.

**GA4 event data that is mapped**

| GA4 event data | Matomo parameter |
|---|---|
| `page_location` | `url` |
| `page_title` | `action_name` |
| `page_referrer` | `urlref` |
| `screen_resolution` | `res` |
| `language` | `lang` |
| `user_agent` | `ua` |
| `client_hints` | `uadata`, with `platform_version` and `full_version_list` renamed to `platformVersion` and `fullVersionList`, the names Matomo reads |
| `ip_override` | `cip` |
| `user_id` | `uid` |

### Matomo client

The incoming hit is forwarded as it is. You can change it with these tables:

- **Parameters to Exclude** / **Parameters to Add / Edit**
- **Custom Dimensions to Exclude** / **Custom Dimensions to Add / Edit**

## Cookies and consent

**Do not use _pk_id for browser ID** (GA4 client):

- `false`: the tag reads the `_pk_id` cookie and sends it as the visitor ID (`_id` and `cid`).
  - If the cookie is missing, a new 16-character ID is created.
  - The cookie is HTTP-only and secure. It is rewritten on every hit, so it expires 400 days after the last visit.
- `true`: no cookie and no visitor ID. Matomo falls back to its own fingerprint (IP, user agent and so on).

The field accepts a variable that returns `true` or `false`, as a boolean or text, for example one built from your consent state. Any other value counts as `true`, so an unclear consent state never sets a cookie.

**Do not use mtm_cart_155c56 for cart management**: turns off the server-side cart cookie. In that case `add_to_cart` and `remove_from_cart` can't be mapped to cart updates.

## Testing

1. Open GTM Server preview and trigger an event.
2. Under **Outgoing HTTP Requests**, open the request to `/matomo.php`. Its body shows the parameters that were sent.
3. Check the response: `{"status":"success","tracked":1,"invalid":0}` means Matomo accepted the hit.
4. Run the tests in the template editor (**Tests** tab).

## License

[Apache 2.0](LICENSE)
