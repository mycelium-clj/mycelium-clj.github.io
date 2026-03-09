# Creating Cells

Cells are the atomic units of Mycelium. Each cell is a pure function with explicit schema contracts.

## Cell Registration

### defcell (recommended)

Register cells via `cell/defcell`. The cell ID is specified once — no duplication:

```clojure
(ns my-app.cells.auth
  (:require [mycelium.cell :as cell]))

(cell/defcell :auth/parse-request
  {:input  [:map [:http-request [:map [:body map?]]]]
   :output {:success [:map [:auth-token :string]]
            :failure [:map [:error-type :keyword]
                           [:error-message :string]]}}
  (fn [_resources data]
    (let [body (get-in data [:http-request :body])]
      (if-let [token (get body "auth-token")]
        {:auth-token token}
        {:error-type :missing-token
         :error-message "No auth token provided"}))))
```

### defmethod (low-level)

The raw multimethod approach requires specifying the ID twice:

```clojure
(defmethod cell/cell-spec :auth/parse-request [_]
  {:id       :auth/parse-request
   :handler  (fn [_resources data]
               (let [body (get-in data [:http-request :body])]
                 (if-let [token (get body "auth-token")]
                   {:auth-token token}
                   {:error-type :missing-token
                    :error-message "No auth token provided"})))
   :schema   {:input  [:map [:http-request [:map [:body map?]]]]
              :output {:success [:map [:auth-token :string]]
                       :failure [:map [:error-type :keyword]
                                      [:error-message :string]]}}
   :requires []})
```

## Cell Spec Fields

| Field | Required | Description |
|-------|----------|-------------|
| `:id` | yes | Keyword identifier, conventionally `namespace/name` (e.g. `:auth/parse-request`) |
| `:handler` | yes | `(fn [resources data] -> data)` for sync, or `(fn [resources data callback error-callback])` for async |
| `:schema` | yes | Map with `:input` (Malli schema) and `:output` (single schema or per-transition map) |
| `:requires` | no | Vector of resource keys the handler needs (e.g. `[:db]`) |
| `:async?` | no | Set to `true` for async handlers |
| `:doc` | no | Documentation string |

## Handler Signature

**Sync** (default):
```clojure
(fn [resources data] -> data)
```
- `resources`: map of injected dependencies (db, http-client, etc.)
- `data`: accumulating data map — all keys from prior cells are present
- Returns: the data map with new keys assoc'd

**Async** (`:async? true`):
```clojure
(fn [resources data callback error-callback])
```
- `callback`: call with `(callback updated-data)` on success
- `error-callback`: call with `(error-callback exception)` on failure

## Output Schema Formats

**Single schema** — all transitions must satisfy it:
```clojure
:output [:map [:result :int]]
```

**Per-transition schemas** — each dispatch transition has its own contract:
```clojure
:output {:found     [:map [:profile [:map [:name :string] [:email :string]]]]
         :not-found [:map [:error-type :keyword] [:error-message :string]]}
```

The transition keys in the output map must match the dispatch labels defined in the workflow's `:dispatches` for this cell.

## Rules

1. Return only new/changed keys — key propagation (on by default) merges input automatically
2. Output must satisfy the declared `:output` schema for the matched transition
3. Never import or call other cells — cells are isolated by design
4. Use only resources passed via the first argument — never acquire external dependencies directly
5. Cell handlers must return a map (not nil) — return `{}` if the cell adds nothing

## Testing Cells in Isolation

```clojure
(require '[mycelium.dev :as dev])

;; Basic test
(dev/test-cell :auth/parse-request
  {:input     {:http-request {:body {"auth-token" "tok_abc"}}}
   :resources {}})
;; => {:pass? true, :output {...}, :errors [], :duration-ms 0.42}

;; With dispatch verification
(dev/test-cell :auth/parse-request
  {:input      {:http-request {:body {"auth-token" "tok_abc"}}}
   :resources  {}
   :dispatches [[:success (fn [d] (:auth-token d))]
                [:failure (fn [d] (:error-type d))]]
   :expected-dispatch :success})
;; => {:pass? true, :matched-dispatch :success, ...}
```

## Common Patterns

### Resource-dependent cell
```clojure
(cell/defcell :user/fetch-profile
  {:input    [:map [:user-id :string]]
   :output   {:found     [:map [:profile [:map [:name :string] [:email :string]]]]
              :not-found [:map [:error-type :keyword] [:error-message :string]]}
   :requires [:db]}
  (fn [{:keys [db]} data]
    (if-let [profile (db/get-user db (:user-id data))]
      {:profile profile}
      {:error-type :not-found
       :error-message "User not found"})))
```

### Async cell (external API call)
```clojure
(cell/defcell :api/fetch-data
  {:input  [:map [:url :string]]
   :output [:map [:response map?]]
   :async? true}
  (fn [_resources data callback error-callback]
    (future
      (try
        (let [resp (http/get (:url data))]
          (callback {:response resp}))
        (catch Exception e
          (error-callback e))))))
```

### Scaffolding cells from a workflow

Use `generate-stubs` to get pre-wired cell definitions from your workflow:

```clojure
(require '[mycelium.dev :as dev])

(println (dev/generate-stubs workflow-def))
;; Prints defcell forms with schemas and TODO handlers — fill in the logic
```
