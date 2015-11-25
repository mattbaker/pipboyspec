# Pipboy App Communication Protocol Specification

This repository endeavors to provide a full description of the protocol used by the Pipboy Companion App in communication to and from Fallout 4. 

## Terms

 * App: the Pipboy Companion App
 * Server: the server that Fallout 4 runs on your PS4, PC, or Xbox One. Also know as "the game."

## Discovery

Before data can be sent and received, the App must discover the Server. See [discovery.md](./discovery.md) for details.

## Communication

Once Discovery has completed, the App connects to the Server on Port TCP/27000. See [communication.md](./communication.md) for details.

## Contributing

This document is based on the work of many. Please submit pull requests as you reverse-engineer new things about the protocol!
