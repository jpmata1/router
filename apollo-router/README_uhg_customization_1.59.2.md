# original repo
    https://github.com/apollographql/router
    
    branch: dev

# forked repo
    https://github.com/duan-HCP/router

    | branch       | vesion(s)       | desc |
    | dev2         | 1.0.5           | the 1st version of customization, incluing extra span labels and configurable metrics buckets |
    | hcp-1.31.0   | 1.31.1          | add "azure_region" labels to more spans |
    | hcp-1.46.0   | 1.46.0          | use built-in metrics buckets, configure extra span labels in yaml file. based on v1.46.0 |
    | hcp-1.53.0   | 1.53.0          | based on v1.53.0 |
    | hcp-1.58.1   | 1.58.1          | based on v1.58.1 |
    | hcp-1.59.2   | 1.59.2          | based on v1.59.2 |

# customization details

0. Checkout branch based on specific tag
```sh
git fetch --all --tags
git checkout -b hcp-1.59.2-customization v1.59.2
```

1. apollo-router/Cargo.toml

```
name = "uhg-custom-appollo-roouter" # HCP customization
version = "1.59.2"

description = "This is a customized Apollo Router, NOT the official apollo router, do not use"
```

2. apollo-router/src/lib.rs

add line #86-88

```
#[allow(missing_docs)]
pub mod uhg_custom;  // HCP customization
```

3. apollo-router/src/uhg_custom.rs --- !!!!! NEW FILE !!!!!

4. fix ```apollo-router``` references in ```cargo.toml``` file **outside** of ```apollo-router``` folders

    files to **exclude** ```./apollo-router```

    replace all ```uhg-custom-appollo-roouter = { ``` with ```uhg-custom-appollo-roouter = {``` 

5. fix ```apollo_router``` references in rust code files (.rs) in ```apollo-router``` folder

    files to **include** ```./apollo-router```, files to **exclude** ```*.md```

    replace all ```use apollo_router::``` with ```use uhg_custom_appollo_roouter::```

    replace all ```apollo_router::main()``` with ```uhg_custom_appollo_roouter::main()```

    replace all ```apollo_router::graphql``` with ```uhg_custom_appollo_roouter::graphql```

    replace all ```apollo_router::services``` with ```uhg_custom_appollo_roouter::services```

    replace all ```apollo_router::TestHarness``` with ```uhg_custom_appollo_roouter::TestHarness```

6. apollo-router/src/plugins/telemetry/metrics/span_metrics_exporter.rs

    add lables (azure_region) to apollo_router_span

    line #71:
    ```
    // HCP customization: add lables azure_region to apollo_router_span
    let azure_region = crate::uhg_custom::get_uhg_azure_region();
    ```

    line #79:
    ```
    // HCP customization - begin!
    // add an extra parameter ", &azure_region" for all record() calls

    e.g: 
    {
        record(duration, "duration", name, Some(&subgraph_name), &azure_region);
        ...
    } else {
        record(duration, "duration", name, None, &azure_region);
        ...
    }
    // HCP customization - end!
    ```

    line #116: (add parameter azure_region)
    ```
    fn record(duration: f64, kind: &'static str, name: &str, subgraph_name: Option<&str>, azure_region: &str) {
        ...
        let attrs = [
            ...

            // HCP customization: add lables azure_region
            KeyValue::new("azure_region", Value::String(azure_region.to_string().into())),
        ];
        ...
    }
    ```

7. apollo-router/src/plugins/telemetry/span_factory.rs

    add lables (azure_region, consumer_name, role_id, correlation_id, cid) to request (REQUEST_SPAN_NAME) span

    line #35:
    ```
    // HCP customization: add lables
    let (azure_region, consumer_name, role_id, correlation_id, cid) = crate::uhg_custom::get_uhg_labels(Some(request.headers()), None);

    ...
    // TODO: Check this as it's not in the v1.58.1
    // HCP customization: add lables azure_region
        KeyValue::new("azure_region", Value::String(azure_region.to_string().into())),
    ...
    ```

    line #48 (error_span), 72 (info_span)
    ```
     error_span!(
        REQUEST_SPAN_NAME,

        // HCP customization - begin!
        "consumerName" = consumer_name,
        "roles" = role_id,
        "correlationId" = correlation_id,
        "cid" = cid,
        "azure_region" = azure_region,
        // HCP customization - end!

        ...
     )
    ```
    (apply the same with info_span)

    add lables (azure_region, consumer_name) to subgraph (SUBGRAPH_SPAN_NAME) span

    line #198:
    ```
    // HCP customization: Add label azure_region and consumer_name to subgraph span
    let (azure_region, consumer_name, _, _, _) = crate::uhg_custom::get_uhg_labels(None, Some(&req.context));
    ```

    lines #225, 238
    ```
     info_span!(
        SUBGRAPH_SPAN_NAME,
        ...

        // HCP customization - begin!
        azure_region = %azure_region,
        consumer_name = %consumer_name,
        // HCP customization - end!
    )
    ```

8. apollo-router/src/services/supergraph/service.rs

    add lables (azure_region, consumer_name) to query planning (QUERY_PLANNING_SPAN_NAME) span

    line #708
    ```
    // HCP customization: add lables
    let (azure_region, consumer_name, _, _, _) = crate::uhg_custom::get_uhg_labels(None, Some(&context));
    ```

    line #722
    ```
    .instrument(tracing::info_span!(
        QUERY_PLANNING_SPAN_NAME,

        // HCP customization - begin!
        azure_region = %azure_region,
        consumer_name = %consumer_name,
        // HCP customization - end!

        "otel.kind" = "INTERNAL"
    ))
    ```

9. apollo-router/src/plugins/telemetry/mod.rs

    backfill "correlationId" for root span

    line #369
    ```
    // HCP customization - begin!
    // backfill "correlationId" for root span, added this for version 1.46.0
    let correlation_id = response.context.get::<_, String>("x-correlation-id");
    if let Ok(Some(correlation_id)) = &correlation_id {
        span.record("correlationId", correlation_id);
    }
    // HCP customization - end!
    ```
    

# publish

```
    cd apollo-router
    cargo install --path . --force
    cargo build
    cargo test
    cargo publish --dry-run
    cargo publish  (version # in apollo-router/Cargo.toml)
```