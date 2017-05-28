#### **Nombre:** Carolina Zúñiga
#### **Código:** A00315292
#### **Url repositorio:** https://github.com/carozuniga/so-project/tree/master/A00315292
# Proyecto Sistemas Operativos

### Configuración de Interfaces

![][1]

### Configuracion de Puertos
Primero se habilita la herramienta ufw que al activarse iniciará en los próximos arranques por defecto

![][2]

Se habilita el puerto 8080 para el funcionaiento de la aplicacion

![][3]

Por último se verifica el estado de los puertos

![][4]

### Instalación de Dependencias
```sh
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
pip install virtualenv
```

Crear un ambiente virtual para el projecto

Para iniciar virtualenvwrapper automáticamente editamos el archivo **.bashrc**

```
vi ~/bashrc
```

Y adicionamos las siguientes líneas al final del archivo

```
export WORKON_HOME=~/.virtualenvs
source /usr/bin/virtualenvwrapper.sh
```

Para usar virtualenvwrapper inmediatamente ejecutamos el siguiente comando desde el usuario python_user

```
source ~/.bashrc
```

Ejecute los siguientes comandos como el usuario python_user, para crear un ambiente virtual e instalar las librerías necesarias.

```
$ mkvirtualenv flask_env
$ pip install flask
$ pip install flask_restplus
```

**Nota:** Al crear un ambiente virtual con virtualenvwrapper, el ambiente creado es automáticamente activado. Esto se puede verificar al observar el nombre del ambiente a la izquierda del prompt ($)

Crear un directorio para el código fuente

```
(flask_env) $ cd ~/
Instalar los requerimientos para el proyecto en el virtualenv.
```

### Aplicación en python

para este punto empleé 3 scripts en python:

**commands.py:** este script tiene los comandos que van a ser usados por check_user y que retornan el uso de la memoria (free -m), el uso del disco (df -h), el uso de la cpu (mpstat), y el servicio escogido sshd (srvice sshd status).

```python
from subprocess import Popen, PIPE

def memory():
 grep= Popen(["free","-m"], stdout=PIPE, stderr=PIPE)
 grep3= Popen(["awk",'/Mem/{print ($3/$2)*100}'],stdin=grep.stdout, stdout=PIPE, stderr=PIPE).communicate()[0]
 return grep3.strip()+'%'

def cpu():
 grep= Popen(["mpstat"], stdout=PIPE, stderr=PIPE)
 grep3= Popen(["awk",'/all/{print $5}'],stdin=grep.stdout, stdout=PIPE, stderr=PIPE).communicate()[0]
 return grep3.strip()+'%'

def disk():
 grep= Popen(["df","-h"], stdout=PIPE, stderr=PIPE)
 grep3= Popen(["awk",'NR == 2{print $5}'],stdin=grep.stdout, stdout=PIPE, stderr=PIPE).communicate()[0]
 return grep3.strip()

def sshd():
 grep= Popen(["service","sshd","status"], stdout=PIPE, stderr=PIPE)
 grep3= Popen(["awk",'/Active/{print $2}'],stdin=grep.stdout, stdout=PIPE, stderr=PIPE).communicate()[0]
 return grep3.strip(
```
**database.py:** este escript realiza el modelo de la base de datos y el esquema de marshmallow. La base de datos consiste en una clase check, con los atributos memory,cpu, disk, sshd, y el identificador.

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_marshmallow import Marshmallow

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/database.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
ma = Marshmallow(app)
db.init_app(app)


class check(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    memory = db.Column(db.String(80),unique=False)
    cpu = db.Column(db.String(80),unique=False)
    disk = db.Column(db.String(80),unique=False)
    ssh = db.Column(db.String(80),unique=False)

    def __init__(self, memory, cpu, disk, ssh):
        self.memory = memory
        self.cpu = cpu
	self.disk= disk
	self.ssh = ssh

    def __repr__(self):
        return '<Checks: memory=%r, cpu=%r, disk=%r, sshd=%r>' % (self.memory,self.cpu,self.disk,self.ssh)

class checkSchema(ma.ModelSchema):
    class Meta:
	model = check
```
**daemon.py:** este es el script que se ejecuta en background y que cada 60 segundos consulta los checks y los guarda en la base de datos, esta base de datos mantiene un tamaño de 100 checks, cuando se llega a 100 se borra el primer check que se hizo para poder guardar uno nuevo.

```python
  from database import db
from database import check
from commands import memory, cpu, disk, sshd
import time

db.create_all()
i=1
while True:

	if (i > 100):
		delete = check.query.first()
		db.session.delete(delete)
		db.session.commit()


	ch= check (memory(),cpu(), disk(), sshd())
	db.session.add(ch)
	db.session.commit()
	i=i+1

	time.sleep(60)

```
La implementación de la aplicación se encuentra en el script app.py

```python
import json
from flask import Flask, abort, request, jsonify
from flask_restplus import Resource, Api
from flask_restplus import fields
from database import check, db, checkSchema
from sqlalchemy import desc
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)

app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/database.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
db.init_app(app)

api = Api(app,version='1.0', title='API for system checks', description='Design and implementation of microservices for the management of infrastructure tasks')


@api.route('/checks')
class ChecksView(Resource):
    @api.response(200, 'Last check successfully returned.')
    def get(self):

	ch = check.query.order_by(desc(check.id)).first()
	return checkSchema().dump(ch).data, 200

    def post(self):        
        return "Does not apply", 501

    def put(self):        
        return "Does not apply", 501

    def delete(self):        
        return "Does not apply", 501

@api.route('/checks/cpu/history')
class cpuHistory(Resource):

	@api.response(200, 'cpu data successfully returned.')
	def get(self):

		size= request.args.get('size')

		ch = check.query.order_by(desc(check.id)).limit(size).all()
		list = {}
		lis = []
		for i in range(len(ch)):
			c = ch[i].cpu
			lis.append(c)

		list["data"] = lis
		return json.dumps(list),200

	def post(self):        
	        return "Does not apply", 501

	def put(self):        
        	return "Does not apply", 501

    	def delete(self):        
        	return "Does not apply", 501

@api.route('/checks/memory/history')
class memoryHistory(Resource):

	@api.response(200, 'memory data successfully returned.')
	def get(self):

		size= request.args.get('size')

		ch = check.query.order_by(desc(check.id)).limit(size).all()
		list = {}
		lis = []
		for i in range(len(ch)):
			c = ch[i].memory
			lis.append(c)

		list["data"] = lis
		return json.dumps(list),200

	def post(self):        
	        return "Does not apply", 501

    	def put(self):        
        	return "Does not apply", 501

    	def delete(self):        
 		return "Does not apply", 501

@api.route('/checks/disk/history')
class diskHistory(Resource):

	@api.response(200, 'disk data successfully returned.')
	def get(self):

		size= request.args.get('size')

		ch = check.query.order_by(desc(check.id)).limit(size).all()
		list = {}
		lis = []
		for i in range(len(ch)):
			c = ch[i].disk
			lis.append(c)

		list["data"] = lis
		return json.dumps(list),200

	def post(self):        
        	return "Does not apply", 501

    	def put(self):        
        	return "Does not apply", 501

    	def delete(self):        
        	return "Does not apply", 501

@api.route('/checks/sshd/history')
class sshdHistory(Resource):

	@api.response(200, 'sshd data successfully returned.')
	def get(self):

		size= request.args.get('size')

		ch = check.query.order_by(desc(check.id)).limit(size).all()
		list = {}
		lis = []
		for i in range(len(ch)):
			c = ch[i].ssh
			lis.append(c)

		list["data"] = lis
		return json.dumps(list),200

	def post(self):        
        	return "Does not apply", 501

    	def put(self):        
        	return "Does not apply", 501

    	def delete(self):        
        	return "Does not apply", 501



if __name__ == "__main__":
    app.run(host='0.0.0.0',port=8080,debug='True')
```

La documentación con swagger es la siguiente:

![][5]

Capturas de pantalla:

Ejecución de daemon.py en background:

![][6]

Prueba de que sí está llenando la base de datos, pasados 7 minutos:

![][7]

Ejecución de app.py:

![][8]

Prueba de la url /checks que retorna el ultimo check realizado (cpu, disco, memoria y sshd) y su identificador:

![][9]

Prueba de la url /checks/cpu/history?size=2, este me retorna el número de los últimos checks de cpu indicado en size:

![][10]

Prueba de la url /checks/memory/history?size=3, este me retorna el número de los últimos checks de memoria indicado en size:

![][11]

Prueba de la url /checks/disk/history?size=1, este me retorna el número de los últimos checks de disco indicado en size:

![][12]

Prueba de la url /checks/sshd/history?size=4, este me retorna el número de los últimos checks de estado de sshd indicado en size:

![][13]




### Referencias
- http://www.websecurity.es/ufw-configura-facilmente-tu-firewall-con-iptables

[1]: Images/interfaces.PNG
[2]: Images/ufwActivation.PNG
[3]: Images/ufwAllow.PNG
[4]: Images/ufwLog.PNG
[5]: Images/apiDoc.png
[6]: Images/daemon.PNG
[7]: Images/database.PNG
[8]: Images/app.PNG
[9]: Images/check.PNG
[10]: Images/cpu.PNG
[11]: Images/memory.PNG
[12]: Images/disk.PNG
[13]: Images/sshd.PNG
