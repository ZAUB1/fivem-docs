---
title: Creating your first script in JS
weight: 412
---
Getting started with scripting for FiveM might be a tad overwhelming, given the wide range of possibilities and the sparsely spread documentation. In this quick and simple guide, we’ll try to show you how to get started with a quick resource in JS. We will be implementing a car spawner through a command.

## Prerequisites
Before creating your first script in JS, there are a couple of things you will need to set up and understand.

* [Understanding of resources and manifest files](/scripting-reference/resource-manifest/resource-manifest)

## Writing code
Now let's write some code !
So we want to implement a command

### Server side
So how do we tell the server that a commands exists ? Well a simple lookup over at https://runtime.fivem.net/doc/natives/ allows you to search for natives and we can quickly find this one :

```c
// 0x5fa79b0f
// RegisterCommand
void REGISTER_COMMAND(char* commandName, func handler, BOOL restricted);
// This native is a void, which means it returns nothing after being called
// We can see in the docs that the different arguments and their types are specified
```
Let's implement it in JS then shall we ?
```js
RegisterCommand("car", (source, args) => { // The handler returns the source and arguments passed though
    // Deal with it
}, false);
```
So we've register a command called car but it still doesn't spawn any car, we need to implement the spawn part. But not too quickly, we are on server side which means we have to trigger something on the player's client to spawn the actual vehicle.
How do we trigger something on the client then ?
```ts
declare function emitNet(eventName: string, target: number|string, ...args: any[]): void
```
As you can see there's a dedicated function available in the FiveM scripting API for this
```js
RegisterCommand("car", (source, args) => {
    const model = args[0] || "osiris"; // Account if no argument is passed

    console.log(model); // We can print the model name
    
    emitNet("spawncar", source, model); // We simply trigger the event
}, false);
```
### Client
You now know how to implent both a command and trigger from server to client, now let's deal with the event we received from server on client side
```ts
declare function on(eventName: string, callback: Function): void
```
This function allows you to listen for an event, following our car example:
```js
on("spawncar", (vehname) => {
    console.log(vehname); // We can also print the model name here
});
```
We still need to make this vehicle spawn though, a quick lookup on the native reference and we find this :
```c
// 0xD80958FC74E988A6 0xFA92E226
// PlayerPedId
Ped PLAYER_PED_ID(); // Returns the ped entity id
```
```c
// 0x1647F1CB 
// GetEntityCoords
Vector3 GET_ENTITY_COORDS(Entity entity); // Returns the coords from an entity
```
```c
// 0xAF35D0D2583051B0 0xDD75460A
// CreateVehicle
Vehicle CREATE_VEHICLE(Hash modelHash, float x, float y, float z, float heading, BOOL isNetwork, BOOL thisScriptCheck);
```
```c
// 0xF75B0D629E1C063D 0x07500C79
// SetPedIntoVehicle
void SET_PED_INTO_VEHICLE(Ped ped, Vehicle vehicle, int seatIndex);
```
Let's finalize our command then :
```js
on("spawncar", (vehname) => {
    RequestModel(vehname); // Loads the model
    
    const ped = PlayerPedId();
    const coords = GetEntityCoords(ped);
    
    const veh = CreateVehicle(vehname, coords[0], coords[1], coords[2], GetEntityHeading(ped), true, false);
    SetPedIntoVehicle(ped, veh, -1); // Puts the player in the driver seat
    
    SetModelAsNoLongerNeeded(vehname) // Releases the model
});
```
