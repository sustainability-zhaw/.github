# SDG Indexing and Dashboard 

This organisation contains services and components for SDG mapping for our tiny interactive Dashboard. 

![Dashboard](https://raw.githubusercontent.com/sustainability-zhaw/.github/main/profile/teaser_screen.png)

The System relies on several micro-services. Each micro service is available as a separate and customizable Docker Container. 

The system is written in primarily in Javascript and Python.

## Core Architecture 

![Service Map](https://raw.githubusercontent.com/sustainability-zhaw/.github/main/profile/services_map.png)

In the above roadmap the blue components are visible to users, while the orange boxes are backend services that run without user interaction. The grey blocks refer to readymade components that are used by the project but contain no business logic. The boxes with dashed grey outline refer to external components that extend the functions and/or data of the system. The boxes with dashed orange outline are data components without business logic. 

Users primarily interact through the Frontend and the index importer via the [keywords](https://github.com/sustainability-zhaw/keywords) repository.

The system uses a joint [data schema](https://github.com/sustainability-zhaw/dgraph-schema) for the data store API. The schema includes also some documentation and sample data for testing directly against the data store.

## Frontend

The frontend is available as a static content stack under the [sustainability-dashboard repository](https://github.com/sustainability-zhaw/sustainability-dashboard).

The frontend is implemented as a single page application that uses the following technologies:

- Core JavaScript ES6 modules. 
- bootstrap5 Sass layouting
- bootstrap5 icons
- poppler for tooltips
- HTML5 with HTML5 templates.

The frontend is designed to rely on minimal external javascript components. It follows the HTML5 principles of separation of structure (markup), layout (stylesheets) and logic (scripts). **This separation is strict**. 

The frontend interacts with the data store via [dgraph's DQL endpoint](https://dgraph.io/docs/dql/). This is necessary because the  GraphQL endpoint cannot provide the complex statistics and filter logic. In order to improve the system performance and security, the datastore has to be protected against the frontend via a caching ACL. 

## Backend

### Data handling

The data is drawn from four resources:

- An [OAI endpoint](http://www.openarchives.org/OAI/openarchivesprotocol.html) (in this case [dspace](https://dspace.lyrasis.org))
- The evento frontend via web-scraping
- The project database via Excel Import
- Other educational catalogs via Excel import

Each importer normalises the data into InfoObjects and stores them in the datastore.

The system distinguishes between *authors* and (internal) *persons*. Persons are equal to institutional members. The system compares the person information from the data sources with the organisational staff directory using the [ad resolver](https://github.com/sustainability-zhaw/ad-resolver). This service identifies different ways of writing names in the data sources and matches them to an institutional member. This automatically builds a set of known pseudonyms, which are otherwise unknown. Also it allows segregation of internal and external contributors. This process will also create an internal model of the active organisational structures. 

All data stores contain messy data, in which elements are placed into inappropriate data fields or not included altogether. The system cleans and extends such information if possible using a data resolver component. Two resolvers are planned:

- Department resolver - fixes departmental assignments based on know contributors affiliations
- Classification resolver - fixes scientific classifications that are misplaced as keywords. 

All data handling services use [dgraph's graphql endpoint](https://dgraph.io/docs/graphql/) to interact with the data store.

### Indexing

The system needs to index all resources based on the provided topics. The indexing uses the `title`, `abstract` and `extra`-information for matching a resource to a topic.

Presently, the indexing is facilitated for SDGs, only.

The indexing will run when:

- index terms were added
- new objects were added to the datastore.

Index terms can be added through the UI's 'index term editor' or via curated spreadsheets from a term repository. 

## Internal Signals

The system uses one relay for all messages. All messages must arrive via the `zhaw-km` exchange. Emitting services must use unique message topics for the information they pass into the system. 

Receiving services must create unique queues in order to receive all messages. Typically, a receiving service will focus on specific message topics.

### Importer Topics

- `importer.oai` - new records from an OAI repository were added (issued by oai extractor)
- `importer.evento` - new records from evento web-scraping were added (issued by by evento extractor)
- `importer.projects` - new records from the project database were added (issued by projects importer)

- `importer.object` - objects have changed in the data store (issued by data store subscription)
- `importer.person` - persons were resolved (issues by ad resolver)

### Indexer Topics

- `indexer.update` - one or more external files have changed (issued by index term importer)
- `indexer.add` - a single term has been added through the UI (issued by data store subscription)

## External Components 

The system consists of front facing components and a backend that runs autonomous. Both parts are glued by a [dgraph](https://dgraph.io) graph database. Given its [graphQL-capabilities](https://graphql.org), the database provides a persistence API to the backend and the frontend. 

All server side components are shielded by a [caddy webserver](https://caddyserver.com). The server provides TLS termination to the internal components and acts as an ingress/reverse proxy to the functional components of the system. 

The access to the dashboard and APIs is secured via OAuth2 using the [authomator service](https://github.com/phish108/authomator). The system leverages on caddy's header introspection capabilities for providing decoupled authentication layer. This allows for seamless decoupling of local, staging and production deployments of the system. 

The backend uses [rabbitMQ](https://rabbitmq.com) as a pub/sub infrastructure for implementing event driven data processing. RabbitMQ allows for horizontal scaling of the functional components. 

## External Signals

The system accepts external web-hooks from GitHub. Each web-hook must configured separately with independent secrets.

Currently, only the index term importer waits for `push` events from the keywords repository.
