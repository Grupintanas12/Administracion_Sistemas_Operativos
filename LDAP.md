dc=telefonica,dc=local

│

├── ou=Informatica

│   ├── cn=Laura Garcia

│   ├── cn=Daniel Lopez

│   └── cn=Pablo Navarro

│

├── ou=RecursosHumanos

│   ├── cn=Marta Sanchez

│   ├── cn=Javier Ruiz

│   └── cn=Elena Castro

│

├── ou=Finanzas

│   ├── cn=Carlos Moreno

│   ├── cn=Lucia Fernandez

│   └── cn=Alberto Gil

│

├── ou=Marketing

│   ├── cn=Sofia Romero

│   ├── cn=Andres Herrera

│   └── cn=Paula Ortega

│

├── ou=AtencionAlCliente

│   ├── cn=Raquel Molina

│   ├── cn=Sergio Vega

│   └── cn=Irene Diaz

│

├── cn=Administradores

└── cn=Empleados


Hemos elegido Telefónica como empresa real. Los nombres de los empleados son ficticios, mientras que los departamentos corresponden a áreas reales de una empresa de telecomunicaciones. Hemos creado el dominio telefonica.local y dentro hemos organizado a 15 empleados en cinco unidades organizativas: Informática, Recursos Humanos, Finanzas, Marketing y Atención al Cliente. También hemos creado dos grupos: Administradores y Empleados. Cada usuario tiene un DN completo que indica su nombre, la OU a la que pertenece y el dominio LDAP.

## DN

### Informatica

cn=Laura,sn=Garcia,ou=Informatica,dc=telefonica,dc=local

cn=Daniel,sn=Lopez,ou=Informatica,dc=telefonica,dc=local

cn=Pablo,sn=Navarro,ou=Informatica,dc=telefonica,dc=local

### RRHH

cn=Marta,sn=Sanchez,ou=RecursosHumanos,dc=telefonica,dc=local

cn=Javier,sn=Ruiz,ou=RecursosHumanos,dc=telefonica,dc=local

cn=Elena,sn=Castro,ou=RecursosHumanos,dc=telefonica,dc=local

### Finanzas

cn=Carlos,sn=Moreno,ou=Finanzas,dc=telefonica,dc=local

cn=Lucia,sn=Fernandez,ou=Finanzas,dc=telefonica,dc=local

cn=Alberto,sn=Gil,ou=Finanzas,dc=telefonica,dc=local

### Marketing

cn=Sofia,sn=Romero,ou=Marketing,dc=telefonica,dc=local

cn=Andres,sn=Herrera,ou=Marketing,dc=telefonica,dc=local

cn=Paula,sn=Ortega,ou=Marketing,dc=telefonica,dc=local

### Atención al Cliente

cn=Raquel,sn=Molina,ou=AtencionAlCliente,dc=telefonica,dc=local

cn=Sergio,sn=Vega,ou=AtencionAlCliente,dc=telefonica,dc=local

cn=Irene,sn=Diaz,ou=AtencionAlCliente,dc=telefonica,dc=local

### Grupos

cn=Administradores,dc=telefonica,dc=local

cn=Empleados,dc=telefonica,dc=local
