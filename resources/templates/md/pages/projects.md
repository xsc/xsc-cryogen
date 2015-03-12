{:title "Projects", :layout :page, :page-index 0, :navbar? true}

### Leiningen Plugins

- __[lein-ancient](https://github.com/xsc/lein-ancient)__ checks your projects
  or whole Leiningen environment for outdated dependencies and automatically
  upgrades them to current versions.
- __[lein-thriftc](https://github.com/xsc/lein-thriftc)__ compiles your Thrift
  IDL files and puts them on the classpath.

### Lifecycle & Components

- __[minion](https://github.com/xsc/minion)__ let's you create entry points for
  your applications, including command-line parsing, embedding an nREPL server,
  etc...
- __[peripheral](https://github.com/xsc/peripheral)__ offers macros for simple
  component and system generation using Stuart Sierra's
  [component](https://github.com/stuartsierra/component) library. It
  additionally handles things like ad-hoc subsystems, graceful rollback after
  initialization failures and some others.

### Utility Libraries

- __[pandect](https://github.com/xsc/pandect)__ is a fast and simple hash,
  checksum and HMAC library.

### Ring & Ronda

- __[ronda/routing](https://github.com/xsc/ronda-routing)__ aims at decoupling
  routing and handlers, allowing for e.g. conditional middlewares, injection of
  route-based request metadata and much more.
- __[ronda/schema](https://github.com/xsc/ronda-schema)__ translates prismatic's
  schema validation to HTTP requests and responses.
