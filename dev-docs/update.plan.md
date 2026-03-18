1. Architectural & Performance Enhancements (Critical)
   * Fix Blocking I/O in the Event Loop: OpenCanary is built on the asynchronous Twisted framework. However,
     the SlackHandler, TeamsHandler, and WebhookHandler in opencanary/logger.py use the synchronous requests
     library inside their emit methods.
       * Impact: When an alert is triggered, these synchronous HTTP requests will block the entire Twisted
         reactor (event loop). If the alert destination API is slow or unreachable, the entire honeypot will
         freeze, leading to denial-of-service (DoS) under load.
       * Suggestion: Replace requests with Twisted's asynchronous HTTP client tools like treq or
         twisted.web.client.Agent to perform non-blocking API calls.

  2. Security Hardening
   * Environment Variable Expansion Risk: In opencanary/config.py, the expand_vars function automatically
     expands environment variables in configuration strings.
       * Impact: While convenient, if the honeypot's configuration values are ever reflected in responses
         (e.g., HTTP banners or custom skins) or logs, it could inadvertently expose sensitive system
         environment variables to an attacker.
       * Suggestion: Consider restricting variable expansion to specific, safe fields, or disable it by
         default unless explicitly enabled.
   * Docker Security (Privilege Dropping): Honeypots often need to bind to privileged ports (like 80 for HTTP
     or 22 for SSH). If the Dockerfile runs the OpenCanary daemon as root to achieve this, a compromise of the
     honeypot could lead to container root access.
       * Suggestion: Ensure the Docker container runs as a non-root user. Use setcap (e.g., setcap
         'cap_net_bind_service=+ep' /usr/local/bin/python) to allow the Python binary to bind to lower ports
         without requiring full root privileges.
   * Resource Constraining: To prevent resource exhaustion attacks against the honeypot itself, ensure that
     the Twisted services have explicit limits on maximum concurrent connections and request sizes.

  3. Reliability & Maintainability
   * Improve Error Handling: The HpfeedsHandler in opencanary/logger.py uses bare except: blocks.
       * Impact: Bare except blocks catch SystemExit and KeyboardInterrupt, and they hide underlying bugs,
         making debugging difficult.
       * Suggestion: Change bare except: to except Exception as e: and log the specific error.
   * Modernize Dependency Management: The project relies on pkg_resources (from setuptools) for resource
     loading.
       * Suggestion: pkg_resources is deprecated in modern Python. Migrate to importlib.metadata and
         importlib.resources for a more standard and performant approach to loading internal data and skins.
   * Refactor HTML Templating: The HTTP honeypot module (opencanary/modules/http.py) relies on manual string
     substitution (e.g., replacing [[URL]] or [[BANNER]]) to serve "fake" admin panels.
       * Suggestion: While functional, manual substitution is rigid and can be prone to injection if inputs
         aren't perfectly sanitized. Introducing a lightweight, secure templating engine like Jinja2 with
         auto-escaping enabled would improve maintainability and security against Cross-Site Scripting (XSS)
         within the honeypot's simulated UI.
                                                    