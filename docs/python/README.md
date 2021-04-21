# Python SSB implementations

[PySecretHandshake](https://github.com/pferreir/PySecretHandshake) implements Secret Handshake as specified in Dominc Tarr's paper ["Designing a Secret Handshake: Authenticated Key Exchange as a Capability System"](http://dominictarr.github.io/secret-handshake-paper/shs.pdf) (Dominic Tarr, 2015). It is required by [pyssb](https://github.com/pferreir/pyssb). 

Things that are currently implemented by pyssb are:

> - Basic Message feed logic
> - Secret Handshake
> - packet-stream protocol

With these the client side is fully functional and one can already use it to communicate with a ssb-server.

