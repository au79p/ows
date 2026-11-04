# Au79 Open Wholesaler Specification

Version 0.1.0 Draft. The machine-readable contract uses OpenAPI 3.2.1.

Au79 Open Wholesaler Specification defines a small HTTP contract for a wholesaler service that provides executable gold prices, customer balances, and order execution. It also defines optional live price delivery and order-status webhooks.

## Files

- [SPEC.md](SPEC.md): normative protocol specification.
- [openapi.yaml](openapi.yaml): machine-readable HTTP contract.
- [examples](examples): minimal request, response, event, and error examples.

## Scope

Version 0.1 requires HTTPS JSON endpoints for capabilities, price snapshots, balances, orders, and order status. Server-Sent Events (SSE) and order-status webhooks are optional capabilities.

WebSocket and provider-specific protocols are outside this specification. An adapter may translate them to this contract.

## License

Documentation and examples are licensed under [CC BY 4.0](LICENSE).
