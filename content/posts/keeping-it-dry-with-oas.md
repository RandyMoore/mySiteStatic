---
title: "Keeping it DRY with OAS"
date: 2019-12-01T00:00:00+00:00
draft: false
---

Don't Repeat Yourself (DRY) is a [well known](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle in software development. An Open API Specification (OAS) is one way to adhere to this principle. An OAS is an example of [declarative programming](https://en.wikipedia.org/wiki/Declarative_programming) - it is data that declares what the Application Programming Interface (API) is. Folks will typically see the benefit of an OAS as providing [a source of documentation for an API](https://swagger.io/tools/swagger-ui/). Surprisingly there are other benefits as well, vertically and horizontally across the stack in a system with a [multi tier architecture](https://en.wikipedia.org/wiki/Multitier_architecture) thanks to the DRYness provided by an OAS.

## Shared Data Definitions
The [components part of an OAS](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#componentsObject) allows for defining data types that can be reused across the stack. [Tools](https://openapi-generator.tech/) can read the declaration and generate code in a [multitude of languages](https://openapi-generator.tech/docs/generators) that can convert data over the wire (e.g. JSON in an HTTP body) into class instances for use in code. The conversion process (serialization and deserialization) may also perform validation based on constraints present in the OAS declaration (e.g. required fields must be present). Having a tool generate the target language code from a single declaration avoids the hell of mismatched definitions across a stack in addition to accelerating the pace of development.

## Endpoint Declaration Benefits
The [paths part of an OAS](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#pathsObject) allows the declaration of what endpoints are available. [Versioning](https://semver.org/) of an OAS may be used to provide assurance of compatibility to clients (both the endpoint and data components). Endpoint paths may be prefixed with the major version of the OAS (e.g. [https://host/v2/endpoint/]()) allowing for the evolution of the API in terms of backwards compatible features and bug fixes while preserving client trust. Tools can generate code for both the client (SDK library) and server sides. [Tools also exist that can validate dynamic behavior between parts of a system in test](https://opensource.com/article/18/6/better-api-testing-openapi-specification) from the OAS declaration and the constraints it imposes.

## The Big Picture
Having a single source of truth in a language agnostic data format opens the door to automation. Development accelerates, opportunities for mistakes are removed. Scalability in terms of coordinating development is super charged. [Micro services architecture](https://en.wikipedia.org/wiki/Microservices) and other loosely coupled system organizations benefit enormously from clearly defined interfaces, enabling the following [Agile Manifesto principle](https://agilemanifesto.org/principles.html):

> The best architectures, requirements, and designs
emerge from self-organizing teams.

## An Example You Can Play With
Here is an example demonstrating generation of a Javascript SDK and a Flask server from an [OAS](https://github.com/RandyMoore/OAS/blob/master/openapi/oas.yaml). It includes a simple demo showing usage of the Javascript SDK to fetch a resource from the server. Additionally, an example is provided for how to generate a stand alone server hosting a Swagger UI page to document the API per the declared OAS.
https://github.com/RandyMoore/OAS