Protocol Todos
--------------
* Include error handling.
* Make a more consistent and stable packet structure.
* Set up a system where components can advertise a list of "features" they handle and a way to search for a feature, and possibly a prefix which routes publishes to all components which handle a feature, though this wouldn't be particularly useful. On hold until I come up with a plausible usecase.
* Define component introspection in the spec, where components have certain actions which they **should** handle and respond to with an informative packet about itself (possibly including the aforementioned feature listing).
* Router linking, maybe even with an IRC/email hybrid where routers can communicate easily and continuously but without a certain router going down affecting more than just the components connected directly to it.
