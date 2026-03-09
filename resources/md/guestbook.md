## Guestbook Application

This tutorial will guide you through building a simple guestbook application using Mycelium.
The guestbook allows users to leave a message and to view a list of messages left by others.
The application will demonstrate the basics of Mycelium workflows, HTML templating, database access, and
project architecture.

If you don't have a preferred Clojure editor already, then it's recommended that you use [Calva](https://calva.io/getting-started/) to follow along with this tutorial.

### Installing JDK

Clojure runs on the JVM and requires a copy of JDK to be installed. If you don't
have JDK already on your system then OpenJDK is recommended and can be downloaded
[here](http://www.azul.com/downloads/zulu/). Note that Mycelium requires JDK 11 or greater to
work with the default settings. Alternatively, follow the instructions for installing packages
on your system.

### Installing a Build Tool

Mycelium uses [Clojure Deps and CLI](https://clojure.org/guides/deps_and_cli) for building and running projects.

To install Clojure CLI, follow the steps below depending on your operating system.

MacOS

```
brew install clojure/tools/clojure
```

Linux

```
curl -L -O https://github.com/clojure/brew-install/releases/latest/download/posix-install.sh
chmod +x posix-install.sh
sudo ./posix-install.sh
```

### Creating a new application

Once you have the Clojure CLI installed, you can run the following commands in your terminal to
initialize your application:

```
clj -Tdeps-new create :template io.github.mycelium-clj/mycelium-web-template :name yourname/guestbook
cd guestbook
```

The above will create a new project, named `yourname/guestbook`, based on the Mycelium web template.

### Anatomy of a Mycelium application

The newly created application has the following structure:

```
guestbook/
├── deps.edn
├── resources/
│   ├── system.edn
│   ├── logback.xml
│   └── html/
│       └── home.html
├── src/clj/yourname/guestbook/
│   ├── core.clj
│   ├── config.clj
│   ├── web/
│   │   ├── handler.clj
│   │   ├── middleware/
│   │   │   └── core.clj
│   │   └── routes/
│   │       └── pages.clj
│   ├── cells/
│   │   └── home.clj
│   └── workflows/
│       └── home.clj
├── env/
│   ├── dev/clj/
│   │   ├── user.clj
│   │   └── yourname/guestbook/
│   │       ├── env.clj
│   │       └── dev_middleware.clj
│   └── prod/clj/
│       └── yourname/guestbook/
│           └── env.clj
└── test/clj/yourname/guestbook/
    └── core_test.clj
```

Let's take a look at what the files in the root folder of the application do:

* `deps.edn` - used to manage the project configuration and dependencies
* `resources/system.edn` - used for system configuration with [Integrant](https://github.com/weavejester/integrant) and [Aero](https://github.com/juxt/aero)
* `.gitignore` - a list of assets, such as build-generated files, to exclude from Git

### The Source Directory

All our code lives under the `src/clj` folder. Since our application is called yourname/guestbook, this
is the root namespace for the project. Let's take a look at all the namespaces that have been created for us.

#### guestbook

* `config.clj` - this is the place where your `system.edn` is read in to create an Integrant configuration map
* `core.clj` - this is the entry point for the application that contains the logic for starting and stopping the server

#### guestbook.web

The `web` namespace is used to define the edges of your application that deal with server communication, such as receiving HTTP requests and returning responses.

* `handler.clj` - defines the Ring handler and router configuration
* `middleware/core.clj` - an aggregate of default middleware and environment-specific middleware
* `routes/pages.clj` - HTTP routes wired to Mycelium workflows

#### guestbook.cells

The `cells` namespace is where Mycelium cell definitions live. Cells are functions with explicit input/output schemas. You can think of them as being akin to microservices. Each one accepts resources and a state map, then returns a new state that propagates in the workflow. These functions can do IO and produce side effects.

* `home.clj` - cell definitions for the default home page

#### guestbook.workflows

The `workflows` namespace is where Mycelium workflow definitions live. Workflows compose cells into directed graphs.

* `home.clj` - the home page workflow (a pipeline of cells)

### Understanding Cells and Workflows

Mycelium applications are built from two core concepts:

**Cells** represent individual steps in a workflow, similar to microservices. Each cell can do IO and produce side effects, and the state it returns is used by the workflow engine to decide which cell to run next. Cells have explicit input/output schemas and are registered via `cell/defcell`, receiving a resources map and a data map:

```clojure
(cell/defcell :request/parse-home
  {:input  [:map [:http-request :map]]
   :output [:map [:name :string]]
   :doc    "Extract name parameter from the HTTP request"}
  (fn [_resources data]
    (let [params (get-in data [:http-request :query-params])
          name   (or (get params "name") "World")]
      {:name name})))
```

The `:schema` map declares what data the cell expects (`:input`) and what it produces (`:output`). These schemas use [Malli](https://github.com/metosin/malli) and are validated at compile time.

**Workflows** compose cells into directed graphs. The simplest form is a `:pipeline`:

```clojure
(def workflow-def
  {:cells    {:start  :request/parse-home
              :render :page/render-home}
   :pipeline [:start :render]})

(def compiled (myc/pre-compile workflow-def))
```

Data **accumulates** through the pipeline — each cell receives all keys from prior cells plus the initial input. The workflow is pre-compiled at load time for optimal performance.

Compiled workflows are wired to HTTP routes using `mycelium.middleware/workflow-handler`:

```clojure
(mw/workflow-handler home/compiled {})
```

The handler passes the Ring request as `{:http-request request}` and extracts the `:html` key from the result as the response body.

### The Env Directory

Environment-specific code and resources are located under the `env/dev` and `env/prod` paths.
The `dev` configuration will be used during development and testing,
while the `prod` configuration will be used when the application is packaged for production.

#### The Dev Directory

* `user.clj` - a utility namespace for REPL development. You start and stop your server from here during development.
* `guestbook/env.clj` - contains the development configuration defaults
* `guestbook/dev_middleware.clj` - contains middleware used for development that should not be compiled in production

#### The Prod Directory

* `guestbook/env.clj` namespace with the production configuration

### The Test Directory

Here is where we put tests for our application. Some test utilities have been provided.

### The Resources Directory

This is where we put all the resources that will be packaged with our application. The `system.edn` file defines the system configuration, and the `html` directory contains Selmer templates.

### Starting Our Server

Your REPL is your best friend in Clojure. Let's start our local development REPL by running

```
clj -M:dev
```

Once you are in the REPL, you can start the system:

```clojure
(go)     ;; Start the system

(halt)   ;; Stop the system

(reset)  ;; Reload code and restart
```

To confirm that your server is running, visit [http://localhost:3000](http://localhost:3000).

### System Configuration

System resources such as HTTP server ports and database connections are defined in the `resources/system.edn` file. For example, this key defines HTTP server configuration:

```clojure
:server/http
 {:port    #long #or [#env PORT 3000]
  :host    #or [#env HTTP_HOST "0.0.0.0"]
  :handler #ig/ref :handler/ring}
```

Components are wired together using Integrant references (`#ig/ref`). The dependency chain is: **server** → **handler** → **router** → **routes**.

Now that we've looked at the structure of the default project, let's add the guestbook functionality.

### Adding Database Support

Our guestbook needs a database to store messages. We'll use SQLite with [next.jdbc](https://github.com/seancorfield/next-jdbc).

First, add the database dependencies to your `deps.edn` under the `:deps` key:

```clojure
com.github.seancorfield/next.jdbc {:mvn/version "1.3.939"}
org.xerial/sqlite-jdbc            {:mvn/version "3.45.3.0"}
```

Next, create a database namespace at `src/clj/yourname/guestbook/db.clj`:

```clojure
(ns yourname.guestbook.db
  (:require [integrant.core :as ig]
            [next.jdbc :as jdbc]))

(defmethod ig/init-key :db/connection [_ {:keys [jdbc-url]}]
  (let [ds (jdbc/get-datasource {:jdbcUrl jdbc-url})]
    (jdbc/execute! ds
      ["CREATE TABLE IF NOT EXISTS guestbook
        (id INTEGER PRIMARY KEY AUTOINCREMENT,
         name VARCHAR(30),
         message VARCHAR(200),
         timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP)"])
    ds))
```

This creates an Integrant component that provides a database connection and ensures the guestbook table exists on startup.

Now add the database configuration to `resources/system.edn`:

```clojure
:db/connection
{:jdbc-url #profile {:dev  "jdbc:sqlite:guestbook_dev.db"
                     :test "jdbc:sqlite:guestbook_test.db"
                     :prod #or [#env JDBC_URL "jdbc:sqlite:guestbook.db"]}}
```

Update the `:reitit.routes/pages` entry to reference the database:

```clojure
:reitit.routes/pages
{:db #ig/ref :db/connection}
```

Finally, require the db namespace in `core.clj` so the Integrant key is registered:

```clojure
(ns yourname.guestbook.core
  (:require
    ...
    [yourname.guestbook.db]))
```

### Creating Guestbook Cells

Now let's define the cells for our guestbook. Create a new file at `src/clj/yourname/guestbook/cells/guestbook.clj`:

```clojure
(ns yourname.guestbook.cells.guestbook
  (:require [mycelium.cell :as cell]
            [next.jdbc :as jdbc]
            [next.jdbc.result-set :as rs]
            [selmer.parser :as selmer]))

(cell/defcell :guestbook/load-messages
  {:input    [:map]
   :output   [:map [:messages [:sequential :map]]]
   :doc      "Load all guestbook messages from the database"
   :requires [:db]}
  (fn [{:keys [db]} _data]
    {:messages (jdbc/execute! db
                 ["SELECT * FROM guestbook ORDER BY timestamp DESC"]
                 {:builder-fn rs/as-unqualified-maps})}))

(cell/defcell :page/render-guestbook
  {:input  [:map [:messages [:sequential :map]]]
   :output [:map [:html :string]]
   :doc    "Render the guestbook page with messages"}
  (fn [_resources data]
    {:html (selmer/render-file "html/guestbook.html"
                                {:messages (:messages data)})}))
```

The `:guestbook/load-messages` cell declares `:requires [:db]`, which means it expects a `:db` key in its resources map. We use `rs/as-unqualified-maps` so that result columns are returned as simple keys like `:name` and `:message` rather than qualified keys like `:guestbook/name`.

The `:page/render-guestbook` cell takes the loaded messages and renders a [Selmer](https://github.com/yogthos/Selmer) template.

### Creating the Guestbook Workflow

Create the workflow at `src/clj/yourname/guestbook/workflows/guestbook.clj`:

```clojure
(ns yourname.guestbook.workflows.guestbook
  (:require [mycelium.core :as myc]
            ;; Load cell definitions
            [yourname.guestbook.cells.guestbook]))

(def home-def
  {:cells    {:start  :guestbook/load-messages
              :render :page/render-guestbook}
   :pipeline [:start :render]})

(def home-compiled (myc/pre-compile home-def))
```

This defines a simple pipeline: load messages from the database, then render the page. Note that the first cell in a pipeline must be named `:start` — this is how Mycelium determines the entry point. The workflow is pre-compiled at namespace load time for optimal performance.

### Adding Routes

Update `src/clj/yourname/guestbook/web/routes/pages.clj` to add our guestbook routes:

```clojure
(ns yourname.guestbook.web.routes.pages
  (:require [integrant.core :as ig]
            [mycelium.middleware :as mw]
            [next.jdbc :as jdbc]
            [yourname.guestbook.workflows.guestbook :as guestbook]))

(defn save-message-handler [db]
  (fn [{:keys [params]}]
    (let [{:keys [name message]} params]
      (when (and (seq name) (seq message))
        (jdbc/execute! db
          ["INSERT INTO guestbook (name, message) VALUES (?, ?)"
           name message])))
    {:status  302
     :headers {"Location" "/"}
     :body    ""}))

(defn page-routes [{:keys [db]}]
  [["/" {:get {:handler (mw/workflow-handler
                          guestbook/home-compiled
                          {:resources {:db db}})}}]
   ["/save-message" {:post {:handler (save-message-handler db)}}]])

(derive :reitit.routes/pages :reitit/routes)

(defmethod ig/init-key :reitit.routes/pages
  [_ opts]
  (fn []
    ["" (page-routes opts)]))
```

The home page route uses `mw/workflow-handler` to wire the compiled workflow to the GET endpoint. The workflow handler automatically passes the Ring request as `{:http-request request}` and extracts the `:html` key from the workflow result as the response body. Resources (in this case, the database connection) are passed through the options map.

The save route uses a simple handler function that extracts the form parameters, saves the message to the database, and redirects back to the home page. We use HTML5 `required` attributes on the form inputs for client-side validation.

### Creating the HTML Template

Create `resources/html/guestbook.html`:

```xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Guestbook</title>
    <link rel="stylesheet"
          href="https://cdn.jsdelivr.net/npm/bulma@0.9.4/css/bulma.min.css">
    <style>
        ul.messages { list-style: none; }
        ul.messages li {
            padding: 10px;
            border-bottom: 1px dotted #ccc;
        }
        li:last-child { border-bottom: none; }
        li time { font-size: 12px; color: #888; }
        form { padding: 30px; }
    </style>
</head>
<body>
<section class="section">
    <div class="container">
        <h1 class="title">Guestbook</h1>

        <div class="columns">
            <div class="column">
                <h3 class="subtitle">Messages</h3>
                <ul class="messages">
                    {% for item in messages %}
                    <li>
                        <time>{{item.timestamp}}</time>
                        <p>{{item.message}}</p>
                        <p><strong>- {{item.name}}</strong></p>
                    </li>
                    {% endfor %}
                </ul>
            </div>
        </div>

        <div class="columns">
            <div class="column">
                <form method="POST" action="/save-message">
                    <div class="field">
                        <label class="label">Name</label>
                        <div class="control">
                            <input class="input" type="text"
                                   name="name" required />
                        </div>
                    </div>
                    <div class="field">
                        <label class="label">Message</label>
                        <div class="control">
                            <textarea class="textarea"
                                      name="message" required></textarea>
                        </div>
                    </div>
                    <div class="field">
                        <div class="control">
                            <input type="submit"
                                   class="button is-primary"
                                   value="Submit" />
                        </div>
                    </div>
                </form>
            </div>
        </div>
    </div>
</section>
</body>
</html>
```

We use a `for` iterator to walk through the messages. Since each message is a map with `:name`, `:message`, and `:timestamp` keys, we can access them by name in the Selmer template.

The form uses HTML5 `required` attributes for client-side validation, and submits a POST request to `/save-message`.

### Testing the Application

Restart your REPL to pick up the new dependencies and code:

```clojure
(reset)
```

Visit [http://localhost:3000](http://localhost:3000) and try adding some messages!

You can also test the database from the REPL:

```clojure
(require '[next.jdbc :as jdbc])
(require '[next.jdbc.result-set :as rs])

(def db (:db/connection integrant.repl.state/system))

(jdbc/execute! db
  ["INSERT INTO guestbook (name, message) VALUES (?, ?)"
   "REPL User" "Hello from the REPL!"])

(jdbc/execute! db
  ["SELECT * FROM guestbook"]
  {:builder-fn rs/as-unqualified-maps})
```

## Adding some tests

Create a test at `test/clj/yourname/guestbook/guestbook_test.clj`:

```clojure
(ns yourname.guestbook.guestbook-test
  (:require [clojure.test :refer :all]
            [next.jdbc :as jdbc]
            [next.jdbc.result-set :as rs]
            [mycelium.core :as myc]
            [yourname.guestbook.cells.guestbook]
            [yourname.guestbook.workflows.guestbook :as guestbook])
  (:import [java.io File]))

(defn test-db []
  (let [tmp  (File/createTempFile "guestbook-test" ".db")
        path (.getAbsolutePath tmp)
        ds   (jdbc/get-datasource {:jdbcUrl (str "jdbc:sqlite:" path)})]
    (.deleteOnExit tmp)
    (jdbc/execute! ds
      ["CREATE TABLE IF NOT EXISTS guestbook
        (id INTEGER PRIMARY KEY AUTOINCREMENT,
         name VARCHAR(30),
         message VARCHAR(200),
         timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP)"])
    ds))

(deftest test-home-workflow
  (let [ds (test-db)
        _  (jdbc/execute! ds
             ["INSERT INTO guestbook (name, message) VALUES (?, ?)"
              "Test User" "Hello World"])
        result (myc/run-compiled
                 guestbook/home-compiled
                 {:db ds}
                 {})]
    (is (= 1 (count (:messages result))))
    (is (= "Test User" (:name (first (:messages result)))))
    (is (string? (:html result)))))
```

Run tests with:

```
clj -M:test
```

## Packaging the application

You can run the application directly using:

```
clj -M:run
```

For building a standalone JAR for deployment, you can add [tools.build](https://clojure.org/guides/tools_build) to your project. Create a `build.clj` at the project root:

```clojure
(ns build
  (:require [clojure.tools.build.api :as b]))

(def lib 'yourname/guestbook)
(def class-dir "target/classes")
(def basis (delay (b/create-basis {:project "deps.edn"})))
(def uber-file "target/guestbook-standalone.jar")

(defn clean [_]
  (b/delete {:path "target"}))

(defn uber [_]
  (clean nil)
  (b/copy-dir {:src-dirs   ["src/clj" "resources" "env/prod/clj"]
               :target-dir class-dir})
  (b/compile-clj {:basis     @basis
                  :src-dirs  ["src/clj" "env/prod/clj"]
                  :class-dir class-dir})
  (b/uber {:class-dir class-dir
           :uber-file uber-file
           :basis     @basis
           :main      'yourname.guestbook.core}))
```

Add a `:build` alias to `deps.edn`:

```clojure
:build {:deps  {io.github.clojure/tools.build
                {:git/tag "v0.10.5" :git/sha "2a21b7a"}}
        :ns-default build}
```

Then build and run:

```
clj -T:build uber
java -jar target/guestbook-standalone.jar
```

By default, the production build uses `jdbc:sqlite:guestbook.db`. You can override this by setting the `JDBC_URL` environment variable.

***

The complete working source for this tutorial is available in the [guestbook example](https://github.com/mycelium-clj/mycelium/tree/main/examples/guestbook). The base project template used to generate new projects is in the [mycelium-web-template](https://github.com/mycelium-clj/mycelium-web-template) repository.
