
### Tool data : 
```
-> allowed parameters : 
id_server bigint
id_query

-> Return table : 
status bit, 
log_msg text, 
start_time timestamp, 
end_time timestamp,
exec_duration time,
rows_affected bigint  
```

### Extras 
```
Agregarle la opcion de que realice una sola conexion y cierre una vez termine de ejecutar todas las querys. 
Agregarle que pueda hacer  fetch. y limiten bien todo 
Agregarle que pueda continuar en caso de que se desconecte de un servidor.
Agregarle la opcion de que genere un log automatico con COPY.
En los SQL que tenga la opcion para verbosity en caso de error 
Al momento de crear la tabla de sql que se cree un id unico y despues se elimine , esto para que permita el paralelo.
El usuario debe estar limitado , ninguna query debe de tardar mas de 20 min

```
