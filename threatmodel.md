# Threat model

It is assumed that an attacker may perform one or more the following actions:

1. Intercept and alter network traffic.
1. Compromise some set of weak crypto algorithms which are supported in some legacy cases.
1. Compromise a repository, including gaining access to use any keys stored on the repository.
1. Compromise a signing key, for example due to malicious action or accidental disclosure by the key owner.
1. Compromise a step in the software supply chain.
   This can happen in many different ways, such as by gaining access to the server, compromising the software used in the step of the supply chain, passing different software to a subsequent step than what was intended, or causing an operator to make an error in a step.

While it is not always possible to protect against all scenarios, the system should to the extent possible mitigate and/or reduce the damage caused by a successful attack, detect the occurrence of an attack and notify appropriate parties, yet remain usable for parties operating the system.
Furthermore, the system should recover from successful attacks in a way that presents low operational overhead and risk to users.
