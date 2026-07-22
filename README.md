# Paprika_RedesFinalP


dc=redes2026,dc=local                    ← raíz del dominio (suffix)
│
├── ou=usuarios          (organizationalUnit)
│   ├── uid=jperez       (inetOrgPerson, posixAccount, shadowAccount)  uidNumber 10001
│   ├── uid=mrodriguez                                                 uidNumber 10002
│   ├── uid=csierra                                                    uidNumber 10003
│   ├── uid=atorres                                                    uidNumber 10004
│   └── uid=lmejia                                                     uidNumber 10005
│
├── ou=grupos            (organizationalUnit)
│   └── cn=usuarios-red  (posixGroup)  gidNumber 10000
│
└── ou=equipos           (organizationalUnit)
    ├── cn=pc-bog        (simpleSecurityObject)  cuenta de equipo
    ├── cn=pc-sma
    ├── cn=pc-cuc
    └── cn=pc-baq

cn=admin,dc=redes2026,dc=local           ← rootDN (administración, fuera de ou=usuarios)
