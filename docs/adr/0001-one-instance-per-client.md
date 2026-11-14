# One Instance per Client

Each Client gets an Instance of its own, with its own sign-ins, saved credentials, Bots and Activity, instead of one shared system that holds every Client. We chose this for v1 because it keeps Clients apart by construction: OpenBot assumes one organization per install, so a shared system would need isolation we would have to build and prove ourselves. The cost is that every Client adds an Instance to host, license and upgrade.
