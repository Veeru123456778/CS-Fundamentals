## REST

Rest is Representational State Transfer means like we are storing data in whatever format in db but we will be returning the data in format what client demands, whether it is json, xml, protobuf. So thats why in http we have a header for Content-Type: application/json that is for the json (client is demanding the reponse in json)
While HTTP is a communication protocol, REST is an architectural design style


## How They Work Together

When you build a RESTful API, you are using HTTP as your transport layer.
• HTTP provides the verbs: GET, POST, PUT, DELETE.
• REST provides the meaning: It dictates that GET /users should strictly fetch data, POST /users should create a new user, and the server shouldn't remember your past requests (statelessness).

