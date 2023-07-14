# Take-Home Assignment (v2)

## Part 1: Architectural feedback

### General thoughts

#### Backend for frontend

All GUIs (e.g. `Support Tool GUI`) could consist of two parts: a frontend and a backend for frontend (BFF, see
[Pattern: Backends For Frontends](https://samnewman.io/patterns/architectural/bff/)). The BFF
hides the user authentication implementation for downstream services, like e.g. `User Service`. Thus, the BFF hides
connections to multiple downstream services for the frontend and may therefore act as a gateway to multiple services.
The connections from the BFF to the downstream services are also secured and the user information could be included
in these requests (e.g. if `OAuth2` is used the delegation could be implemented with
[OAuth 2.0 Token Exchange - RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693)).

#### HTTPS

Although in all diagrams `REST` connections are described as `HTTP` I assume secure connections with `HTTPS`.

#### API Gateway to other applications

If other applications from other teams need access to any of the data processed within the `Integration Layer`, a
general API gateway could be added to the `Integration Layer`. This API gateway would act as a general entry point
which delegates/proxies to the various services within the `Integration Layer`.

### Diagram `Architecture 1`

#### Likes

Dedicated GUIs for users with different roles decouples the different use cases.

The `User Store` is only accessible by one service: `User Service`.

For billing there is an own service which decouples billing related processing.

#### Dislikes

Between the `Mail Platform` and the `Integration Layer` a vendor specific protocol is used. The `Integration Layer`
implements this protocol for both the `Support Tool GUI` and the `User Service`. This is not recommended because any
changes in the vendor specific protocol may lead to changes in both components. Thus, if the `Mail Platform` is
replaced, both components needs to be updated accordingly. This is solved in the architecture represented in
diagrams `Architecture 2` and `Architecture 3` where an intermediate service called `Platform Service` abstracts the
vendor specific protocol. This means that any changes or replacement of the `Mail Platform` leads only to changes
in one component - at best, downstream services must not be changed at all.

Since sending an invoice to a customer is not time-critical, the connection from the `User Service` to the
`Billing Service` could be made asynchronous via a message broker like e.g. `RabbitMQ`. This would lead to better
performance, increased reliability, granular scalability and simplified decoupling, see
[Benefits of Message Queues](https://aws.amazon.com/message-queue/benefits/). With a message broker in place, the `SOAP`
connection to the `Billing Service` could be removed which indicates a legacy implementation/flow anyway.

The `Billing Service` possibly needs its own storage, a `Billing Store`, to persist billing relevant data, e.g.
invoice status etc. In order to manage billing relevant data a `Billing GUI` could be added.

As shown in diagrams `Architecture 2` and `Architecture 3` it makes sense to decouple user related data from pricing
related data by introducing a `Pricing Service` with a dedicated `Pricing Store`.

### Diagram `Architecture 2`

#### Likes

Dedicated GUIs for users with different roles decouples the use cases.

For billing there is an own service which decouples billing related processing.

The `Platform Service` abstracts the downstream service(s) of the `Email platform` (reasons see above) to the
`Integration Layer`.

The `Support Tool GUI` is not connected to the `Mail Delivery Agent` directly.

The `Pricing Service` and its dedicated `Pricing Store` decouples user related service and storage (reasons see above).

#### Dislikes

The same dislikes from above regarding missing asynchronous communication to the `Billing Service`, its legacy `SOAP`
connection, the missing `Billing Storage` and `Billing GUI`. Thus, the `Billing Service` needs a connection
to the `User Service` in order to access the relevant data directly.

The `User Config GUI` must communicate to the service it belongs: `User Service`. The hop through the `Platform Service`
to the `User Service` implies that the `Platform Service` concerns too much about user specific data and flows. The
`User Service` provides CRUD on user specific data and configuration, and therefore the `User Config GUI` must
connect to it directly.

The `Pricing Service` must not connect to the `User Store`. CRUD operations on the `User Store` are done in the
`User Service`, therefore a connection between these two services must be implemented in order to remove the connection
between the `User Store` and the `Pricing Service`. The connection from the `Pricing Service` to the `Billing Service`
can be removed.

### Diagram `Architecture 3`

#### Likes

Most of the dislikes from the previous diagrams `Architecture 1` and `Architecture 2` are solved.

The architecture takes multiple concerns into account by separating them accordingly.

#### Dislikes

The same dislikes from above regarding missing asynchronous communication to the `Billing Service`, its legacy `SOAP`
connection, the missing `Billing Storage` and `Billing GUI`.
