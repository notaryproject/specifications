## Threat model
It is assumed that an attacker may perform one or more the following actions:
1. intercept and alter network traffic
2. compromise some set of crypto algorithms, especially known weak algorithms which are supported for legacy purposes
3. compromise a repository, including gaining access to use any keys stored on the repository
4. compromise a signing key, for example due to malicious action or accidental disclosure by the key owner
5. compromise a step in the software supply chain.  This can happen in many different ways, such as by gaining access to the server, compromising the software used in the step of the supply chain, passing different software to a subsequent step than what was intended, or causing an operator to make an error in a step. 

### Security goals
The overarching goal of the system is to maintain the maximum amount of security possible despite the scenario.  In other words, in situations where an attack is successful, it is desirable to do the following:
1. mitigate the damage from the attack.  For example, a successful attack should be contained to as small of a scope as is possible.
2. detect an attack.  For example, if the metadata is tampered with on the repository, when possible this should be detected and reported.
3. prevent an attack.  For example, if an attacker attempts to rollback metadata on the repository to earlier versions, the old metadata must not be trusted by clients.
4. recover from an attack without losing security.  For example, if an attacker compromises a key on a repository, it must be possible to revoke trust in a key and trust a new key instead.  Furthermore, this act of adding trust in a new key should not be done by trusting the old key that is being revoked, because in this case the attacker could potentially also be the one performing the action.
5. maintain usability.  The resulting system is operated in pratice and at scale.  The burden must not be large or it will not be used, especially by users further removed from domain experts.  For example, it seems more desirable to have a Notary repository maintainer need to perform a day's worth of operational key management a year than to have every user need to perform 5 seconds of work.
6. clarity about security provided.  The actual security properties of the system should be understandable to operators, developers and users.  It should be clear to what extent different attacks are covered by the above cases and what the operational action needed is in these situations.
