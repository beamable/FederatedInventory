# Federated Inventory
Beamable supports custom inventory federation using managed [microservices](https://docs.beamable.com/docs/microservices-feature-overview). You can use this feature to extend the Inventory system with items that are managed externally.
  
Some use cases:
- Crypto assets - use NFTs as inventory items and auto-mint new items
- Generative AI - add support for granting players AI-generated items

**Requirements:** 
- Federated Inventory depends on the implementation of [federated Authentication](https://github.com/beamable/FederatedAuthentication). That means that every player first needs to have a federated identity with the same microservice that federates inventory.
- Inventory items are content-driven. To enable federation, content items must be marked as federated to a specific microservice.


## Get inventory items flow with federation enabled
![Get inventory](Diagrams/fetch-inventory-federation-flow.png)

## Grant inventory items flow with federation enabled
![Grant inventory](Diagrams/put-inventory-federation-flow.png)

## Coding guide with examples
### 1. `IFederatedInventory<T>` interface
You should start by implementing the `IFederatedInventory<T>` interface in your microservice. Be aware that this interface also implements `IFederatedLogin<T>` because federated authentication is a prerequisite, as mentioned earlier. `T` must be your implementation of the `IThirdPartyCloudIdentity` - a very simple interface that requires you to define a unique name/namespace for your federation. This enables you to have multiple federation implementations in a single microservice.


```csharp
public class MyFederationIdentity : IThirdPartyCloudIdentity
{
	public string UniqueName => "my-cool-federation";
}

[Microservice("MyFederation")]
public class MyFederationService : Microservice, IFederatedInventory<MyFederationIdentity>
{
    public Promise<FederatedAuthenticationResponse> Authenticate(string token, string challenge, string solution)
    {
        throw new System.NotImplementedException();
    }

    public Promise<FederatedInventoryProxyState> GetInventoryState(string id)
    {
        throw new System.NotImplementedException();
    }

    public Promise<FederatedInventoryProxyState> StartInventoryTransaction(string id, string transaction, Dictionary<string, long> currencies, List<FederatedItemCreateRequest> newItems, List<FederatedItemDeleteRequest> deleteItems, List<FederatedItemUpdateRequest> updateItems)
    {
        throw new System.NotImplementedException();
    }
}
```

### 2. `GetInventoryState` implementation
You can do any custom logic here. For example, you could AI generate some items, load items from a smart contract, use microstorage, or do anything that satisfies your specific requirements.
Here's a dummy example that will return some static items and currency, just to showcase the response structure:

```csharp
public Promise<FederatedInventoryProxyState> GetInventoryState(string id)
{
    return Promise<FederatedInventoryProxyState>.Successful(
        new FederatedInventoryProxyState
        {
            currencies = new Dictionary<string, long>
            {
                { "currency.federated-gold", 1000 },
                { "currency.federated-silver", 5000 }
            },
            items = new Dictionary<string, List<FederatedItemProxy>>
            {
                {
                    "items.avatar", new List<FederatedItemProxy>
                    {
                        new()
                        {
                            proxyId = "externalAvatarId1",
                            properties = new List<ItemProperty>
                            {
                                new()
                                {
                                    name = "level",
                                    value = "20"
                                },
                                new()
                                {
                                    name = "color",
                                    value = "blue"
                                }
                            }
                        },
                        new()
                        {
                            proxyId = "externalAvatarId2",
                            properties = new List<ItemProperty>
                            {
                                new()
                                {
                                    name = "level",
                                    value = "30"
                                },
                                new()
                                {
                                    name = "color",
                                    value = "red"
                                }
                            }
                        }
                    }
                }
            }
        }
    );
}
```

The important thing to emphasize here is the `id` argument. It's the same external user id that you return from the `Authenticate` method. If you want to access the player's id in the Beamable system, you can use `this.Context.UserId`
As an example, you can use the wallet address as an external user identifier when implementing blockchain federation.

### 2. `StartInventoryTransaction` implementation
The Inventory service will forward all the changes against federated currency and items. This method has the same return type as the previous one.
```csharp
public Promise<FederatedInventoryProxyState> StartInventoryTransaction(string id, string transaction, Dictionary<string, long> currencies, List<FederatedItemCreateRequest> newItems, List<FederatedItemDeleteRequest> deleteItems, List<FederatedItemUpdateRequest> updateItems)
{
    await _myFederation.ApplyCurrency(currencies);
    await _myFederation.AddItems(newItems);
    await _myFederation.UpdateItems(updateItems);
    await _myFederation.DeleteItems(deleteItems);
    return await GetInventoryState(id);
}
```

The `transaction` argument is a unique transaction id generated in our Inventory service, and you can use it to guard against multiple submissions.

If your transaction processing is to slow to return a timely response, you can implement the async approach. Process the transaction in the background and just return the current state. Once the transaction finishes processing, you can report back the new state like this:
```csharp
await Requester.Request<CommonResponse>(Method.PUT, $"/object/inventory/{_userContext.UserId}/proxy/state", newState);
```
The Inventory service will notify the game client to refresh the inventory content if there's a diff.

## Beamable provided examples
- [Genamon - a game that uses Generative AI and blockchain with federated authentication and inventory](https://github.com/beamable/genamon-polygon)
- [Polygon/Ethereum authentication and inventory federation](https://github.com/beamable/polygon-example)
- [Solana/Phantom authentication and inventory federation](https://github.com/beamable/solana-example)