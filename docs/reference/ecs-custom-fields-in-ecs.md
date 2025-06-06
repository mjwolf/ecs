---
mapped_pages:
  - https://www.elastic.co/guide/en/ecs/current/ecs-custom-fields-in-ecs.html
applies_to:
  stack: all
  serverless: all
---

# Custom fields [ecs-custom-fields-in-ecs]

ECS defines fields, their datatypes and their usage, and classifies them in "core" and "extended" levels.

However, ECS does not define anything about custom fields. By definition, they are additional fields, exactly as the user or the integration defines them, independently of ECS.

Users and integrations are welcome to capture additional information in their events, as custom fields. This flexibility is by design, and ensures that no one is ever blocked by something not being supported by ECS yet.

ECS is under active development, however. Adding custom fields carries a small risk of conflicting with a future ECS field. There are ways of modeling custom fields that will lead to lower chances of conflict with future versions of ECS. This section outlines a few of these strategies.


## Modeling to reduce chances of conflict [_modeling_to_reduce_chances_of_conflict]


### The `labels` field [_the_labels_field]

Any time a data source has a few extra fields that can be modeled with the `keyword` data type, the simplest way to capture them is with the ECS field `labels`.

Example:

```json
{ "labels": { "foo_id": "beef42", "env": "production" },
  "message": "...",
  "event": { ... }
}
```

If `labels` doesn’t work for your use case, here’s a few more tips to avoid conflicts.


### Proper names [_proper_names]

ECS tries to model information by using the name of concepts, and avoids proper names such as tool names, project names or company names. By extension, nesting custom fields under a proper name is a relatively safe approach to adding custom fields. This is the approach taken by Filebeats modules, for example.

As an example, an HTTP log from HAProxy will contain typical HTTP information, as well as proxy details and statistics. The standard HTTP information can be captured in the ECS field sets `http` and `url`, and the extra details in a custom `haproxy` section:

```json
{ "http": { "request": { "method": "get", ... },
            "response": { "status_code": 200, ... } },
  "url": { "original": "/favicon.ico", ... },
  "haproxy": { "frontend_name": "myfrontend", "backend_name": "mybackend_prod",
               "backend_queue": 0, ... }
}
```


### Capitalization [_capitalization]

ECS strives for a consistent feel by using nesting to group related concepts, and underscores to join words. Following these guidelines for custom fields ensures you preserve the same consistent experience throughout your schemas.

Note however, that breaking away from these guidelines for your custom fields can be used to your advantage. It can be a good way to differentiate between ECS fields and custom fields. Since ECS doesn’t use capitalization for field names, this approach virtually guarantees that custom fields will not conflict with future ECS fields.

Common proxy concepts could modelled via a capitalized, but generic concept name.

HAProxy example:

```json
{ "http": { "request": { "method": "get", ... } },
  "url": { "original": "/favicon.ico", ... },
  "Proxy": { "FrontendName": "myfrontend", "BackendName": "mybackend_prod" },
  "event": { "module": "haproxy" }
}
```

NGINX example:

```json
{ "http": { "request": { "method": "get", ... } },
  "url": { "original": "/favicon.ico", ... },
  "Proxy": { "FrontendName": "another_frontend", "BackendName": "another_backend_prod" },
  "event": { "module": "nginx" }
}
```

The above demonstrates that using a common concept name in custom fields can still be beneficial to correlate among multiple sources that populate them. Using capitalization ensures a future version of ECS that defines a `proxy` field set will not conflict.

Here’s a sample event, during a migration from the custom field, to using a new equivalent ECS field set:

```json
{ "http": { "request": { "method": "get", ... } },
  "Proxy": { "FrontendName": "myfrontend", "BackendName": "mybackend_prod" },
  "proxy": { "frontend_name": "myfrontend", "backend_name": "mybackend_prod" }
}
```

The above will look strange during the migration. However the ability to start populating ECS fields while custom fields are still present in your events makes it possible to decouple the upgrade to a new version of ECS from the time you adjust your pipelines and analysis content.

