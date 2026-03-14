---
layout: post
title:  Broken Object-Level Authorization Bugs with Rust
categories: [API Security]
---

BOLA vulnerability occurs when an API does not check user permissions or is missing authorization checks. BOLA can be considered the API version of the IDOR vulnerability.

Attackers can manipulate API requests and replace the object ID in the endpoint. As a result, they may gain access to another user's data.

Vulnerable code with Rust:

```rust
use axum::{
    Json, Router,
    extract::Path,
    routing::get,
};
use serde::Serialize;

#[derive(Serialize)]
struct Order {
    object_id: u32,
	owner: u32,
    order: String,
}

async fn order_list(Path(order_id): Path<u32>) -> Json<Order> {
    let orders = vec![
        Order {
            object_id: 1,
			owner: 1,
            order: "item1".into(),
        },
        Order {
            object_id: 2,
			owner: 2,
            order: "item2".into(),
        },
    ];

    let order = orders
        .into_iter()
        .find(|x| x.object_id == order_id)
        .expect("ID not found");
    Json(order)
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/api/order/{id}", get(order_list));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

```


There isn't any check for the user authorization, so the api just gets input and shows items.


# Mitigation

**Check the object's owner and current user id**

```rust
   let order = orders
        .into_iter()
        .find(|x| x.object_id == order_id & x.owner == user_id)
	match order{
	   Some(order) => Ok(Json(order)),
	   None => Err("Not Authorized")
	}
```


**Use random identifiers and mitigate enumeration risk**

```rust
GET /api/order/664c3490-8e26-94s8

use uuid::Uuid;

struct Order {
    id: Uuid,
}

```
