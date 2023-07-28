# Federated Inventory
Beamable supports custom inventory federation using managed [microservices](https://docs.beamable.com/docs/microservices-feature-overview). You can use this feature to extend the Inventory system with items that are externally provided and managed.
  
Some use cases:
- Crypto assets - use NFTs as inventory items and auto-mint new items
- Generative AI - add support for granting players AI generated items

**Requiremants:** 
- Federated Inventory depends on the implementation of [federated Authentication](https://github.com/beamable/FederatedAuthentication). That means that every player first needs to have a federated identity with the same microservice that federates inventory.
- Inventory items are content-driven. To enable federation, content item must be marked as federated to a specific microservice.


## Get inventory items flow with federation enabled
![Get inventory](Diagrams/fetch-inventory-federation-flow.png)

## Grant inventory items flow with federation enabled
![Grant inventory](Diagrams/put-inventory-federation-flow.png)