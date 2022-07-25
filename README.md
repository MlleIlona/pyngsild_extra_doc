# Overview
This content is an extra tutorial of *pyngsild tutorial* and *ngsildclient tutorial* done by Fabien BATTELLO.

You can find the full tutorial here :
https://github.com/Orange-OpenSource/pyngsild

Another tutorial is available on this link :
https://pixel-ports.github.io/pyngsi-tutorial.html
(be careful : this second tutorial is obsolete because it was made for pyngsi v2, not pyngsild. However, it is very useful because it is more detailed and the architecture is pretty much the same).

pyngsild works with ngsildclient, the tutorial is here :
https://ngsildclient.readthedocs.io/en/latest/build.html

The aim of this tutorial is to explain simply how to begin the learning process of pyngsild and highlight the difficulties encountered ! 

It is made by a beginner (Ilona GERARD) for beginners !

# What are pyngsild and ngsildclient?

**Pyngsild** is a Python data-centric framework, in Python it is represented by a library. It helps to create NGSI-LD agents.
You can see it as a set of molds to handle data. In a mold (an agent) there is an input (the source) and an output (the sink). According to your needs, the input and the output can vary from one mold to another. 

Pyngsild uses **ngsildclient** to create the agents. The process between the sink and the source is the build of the entities, it is well explained in the link above. In our example, an entity is a material among many others in the mold to create the structure of the agent.

Check the [How it Works](https://github.com/Orange-OpenSource/pyngsild#how-it-works) part of the tutorial, it is well explain !

# Installation - Docker
Instead of using a virtual environment such as pyenv [detailed in pyngsild tutorial](https://github.com/Orange-OpenSource/pyngsild#installation), here we will use a [Docker](https://docs.docker.com/get-started/overview/).

Both has their own advantages and problems, you can choose whatever you prefer.

[Here](https://code.visualstudio.com/docs/containers/quickstart-python#:~:text=Open%20the%20project%20folder%20in%20VS%20Code.%20Open,Flask%2C%20or%20Python%3A%20General%20as%20the%20app%20type.) is a short tutorial to install docker on your computer.

A useful command to launch your docker :
```
docker exec -it [repertory where you docker is] /bin/bash
```

My dockerfile :

```
FROM node
EXPOSE 8080
WORKDIR /app
COPY . /app/
RUN npm install
ENV PROXY_LISTEN_PORT=8080
ENV PROXY_LISTEN_IP=0.0.0.0
ENV ORION_API_URL=http://172.29.1.101:1026/ngsi-ld/v1
ENV MINTAKA_API_URL=http://172.29.1.103:8080
ENV BASE_PATH_NGSILD=/ngsi-ld/v1
ENTRYPOINT ["npm", "start"]
```

# Getting started 

This part is an additional part of the [pyngsild tuto](https://github.com/Orange-OpenSource/pyngsild#getting-started), please refer to it.

To illustrate my purpose, we are going to create an agent that take in input a [.csv file](https://transport.data.gouv.fr/resources/18641/download) from the french [Consolidated national database of carpooling locations (BNLC)](https://transport.data.gouv.fr/datasets/base-nationale-des-lieux-de-covoiturage/) and push it into an API called [OffStreetParking](https://swagger.lab.fiware.org/?url=https://smart-data-models.github.io/dataModel.Parking/OffStreetParking/swagger.yaml). You can check the github of the API [here](https://github.com/FIWARE/data-models/tree/master/specs/Parking).  

Here is our aim :

Collect all the data of the csv file and feed him to the OffStreetParking API (OSP API). We may encounter some problems that we will try to raise in this tuto.

First, all the columns of the BNLC csv do not have a correspondence with an attribute of OffStreetParking API. For example, the *“insee”* code available in the BNLC do not have any equivalent in the OffStreetParking which mean that we will have to complete the API with *new attributes*.

Another important thing is to understand what we are calling, where do you want to get the info ? In our example it is relatively accessible because we just have to split all the columns of the file but in some cases like getting the info from another API, it can be more complicated. We will illustrate it with an example later.

The agent creation could be done differently by calling [the API](https://transport.data.gouv.fr/api/datasets/5d6eaffc8b4c417cdc452ac3) of the BNLC available of the on their website, but especially for csv files, it is easier to split the column to create our Entity.

To reach the Graal of the agent, it is really important to understand the object you are going to manipulate.

## Here is a good recipe for our example
We are going to dissociate the different parts of the code to understand what we are using and we will see different ways to get the data.
- The libraries
- Connect the sink *(output)*
- The conversion of the data *(input)* in .csv format into NGSI-LD format
- The creation of the agent

Dealing with Python, Docker and JavaScript can be hard to manage at first but after practicing it is always the same thing.

## The libraries
Install the libraries on your docker. Your Python version must be on 3.10.+ to use pyngsild.
```python
pip install pyngsild
```
```python
pip install ngsildclient
```
You will always need to import theses libraries, I recommend to import the whole library for an easier use. 
```python
from pyngsild import * 
from ngsildclient import *
```
Here are the other libraries you will need for this tutorial
```python
from pyexpat.errors import XML_ERROR_ATTRIBUTE_EXTERNAL_ENTITY_REF
from pyngsild.agent.bg.http_upload import HttpUploadAgent
from geojson import Point


```
You have everything now to create your agent !

## Connect to the sink

There is 3 different way to get a sink:
- Create your own sink [(here)](https://github.com/Orange-OpenSource/pyngsild#create-a-source)

- Connect to a pyngsild sink already created [(here)](https://pixel-ports.github.io/pyngsi-tutorial.html#Chapter-5-:-Datasources) (the option we are going to do). Here we use the Sinkngsi() function which is connected on the 8080 port to a proxy created for the use of pyngsild called “proxyld”.
```python
sink = SinkNgsi("proxyld", 8080)
```
- Use the SinkStdout() function (very useful when you do not have a [Context Broker](https://www.linkedin.com/pulse/what-context-brokers-alvaro-martin))
```python
sink = SinkStdout()
```

## Function Build Entities
As we said, to convert the date received in input we need to create the material that will fit in the output. This is the role of the function *Build Entities*.

Let’s see how it is laid out !
```python
def build_entity(row: Row) -> Entity:
	id_lieu, id_local, nom_lieu, ad_lieu, com_lieu, insee, type, date_maj, ouvert, source, Xlong, Xlat, nbre_pl, nbre_pmr, duree, horaires, proprio, lumiere, comm = row.record.split(",")
	e = Entity("OffStreetParking", id_lieu.strip('"')) 

	#Attributes already existing in OffStreetParking
	e.tprop("observedDateTime", date_maj.strip('"') + "T11:00:00Z")
	e.prop("totalSpotNumber", nbre_pl.strip('"'))
	e.prop("adress", ad_lieu.strip('"') + " " + com_lieu.strip('"') + " " + nom_lieu.strip('"'))
	e.gprop("location", Point((float(Xlong.strip('"')) ,float(Xlat.strip('"')))))
	e.prop("maximumParkingDuration", duree.strip('"'))
	e.prop("owner", proprio.strip('"'))
	e.prop("openingHours", horaires.strip('"'))
	e.prop("description", comm.strip('"'))

	#New Attributes
	e.prop("insee", insee)
	e.prop("id_local", id_local)
	e.prop("type", type)
	e.prop("ouvert", ouvert)
	e.prop("source", source)
	e.prop("nbre_pmr", nbre_pmr)
	e.prop("lumiere", lumiere)

	return e
### Create the function
```
Let’s analyze the code.

```python
def build_entity(row: Row) -> Entity:
```
Here we use an pyngsild object called *Row*. It is our data coming in our agent, the one with we will build an NGSI-LD *Entity* from it.

There are two attributes to this object 
- record → the raw incoming data, it can be *any type*
- provider → the label of the data, it is a *string type*

```
"id_lieu","id_local","nom_lieu","ad_lieu","com_lieu","insee","type","date_maj","ouvert","source","Xlong","Ylat","nbre_pl","nbre_pmr","duree","horaires","proprio","lumiere","comm"
"2A004-C-001","","A Confina","Lieu-dit Stagnacciu (Croisement RT22 - RD31)","AJACCIO","2A004","Supermarché","2019-08-28","true","539830349","8.783403","41.9523692","0","0","","24/7","Prive","true","transport : Bus Muvistrada (2)"
"01024-C-001","","Mairie d'Attignat","Place De La Fontaine- Anciens Combattants","ATTIGNAT","01024","Parking","2019-08-29","true","200071751","5.158352778","46.28957222","5","","","","","",""
"01049-C-001","","Aire de covoiturage Montluel","La Boisse","LA BOISSE","01049","Aire de covoiturage","2019-08-29","true","240100610","5.048432","45.840543","100","","","","","",""
```
This is how is made our csv file. Each carshare location has an **”id_lieu”**, an **”id_local”**, etc. If the cell is empty there is only a quote mark “ ”.

### Split the csv columns
```python
id_lieu, id_local, nom_lieu, ad_lieu, com_lieu, insee, type, date_maj, ouvert, source, Xlong, Xlat, nbre_pl, nbre_pmr, duree, horaires, proprio, lumiere, comm = row.record.split(",")
```
In our example, we want to get the info of each column separately to feed them in the right attribute of OffStreetParking. For that we use the *.record* to get the incoming data and *.split()* function to separate each column.

You can illustrate each column of the csv as a box with a name such as “nom_lieu”. Each box contain the information of all the line of the csv. It means that “nom_lieu” has in his box all the location names of the carshare location.

Also, you can see that the columns on the csv are separated by a comma. Sometimes it can be separated by a semi-colon, it depends of the file, it is a common error.

### Create the Entity
```python
e = Entity("OffStreetParking", id_lieu.strip('"')) 
```
We use the *Entity()* function of *ngsildclient*. We create the entity of the ‘OffStreetParking’ API and the main key will be id_lieu (this is the key written on the doc on the BNLC website)

To call an attribute of a csv file we just have to call it. The *strip()* function use after it is to pull the quote out. We will do it each time we call an argument, otherwise we also get it and it can create bugs mostly when we are not pushing in the api strings. We manipulate other Python or Pyngsild types like floats, integers, tuples, **geoProperty**, etc.

```python
#Attributes already existing in OffStreetParking
	e.tprop("observedDateTime", date_maj.strip('"') + "T11:00:00Z")
	e.prop("totalSpotNumber", nbre_pl.strip('"'))
	e.gprop("location", Point((float(Xlong.strip('"')) ,float(Xlat.strip('"')))))
```
Let’s focus now on [ngsildclient **properties**](https://ngsildclient.readthedocs.io/en/latest/build.html#add-properties). It is well explained on the main doc so I recommend you to read it.

You have to call the property with the same name as the output.

You can find the properties you will need on the [OffStreetParking swagger](https://swagger.lab.fiware.org/?url=https://smart-data-models.github.io/dataModel.Parking/OffStreetParking/swagger.yaml). For example, to feed the attribute ‘location’ we need coordinates, for that you must *import GeoJson Point* (imported at the beginning of the doc ). We can see in the columns name of the csv that we have *Xlong* and *Xlat* attributes. Perfect ! We can give theses coordinates to the attribute location. Because it is a GeoProp we use the [*gprop()*](https://ngsildclient.readthedocs.io/en/latest/build.html#geoproperty) function. 

```python
#New Attributes
	e.prop("insee", insee)
	e.prop("id_local", id_local)
	e.prop("type", type)
	e.prop("ouvert", ouvert)
	e.prop("source", source)
	e.prop("nbre_pmr", nbre_pmr)
	e.prop("lumiere", lumiere)

	return e
```

When you do not have a correspondence between the input and the output you have to create your own output attribute. In this example, there was not any ‘lumiere’ attribute in the OffStreetParking API so I created it.

At the end of your function you return your new Entity and that’s it !


The Build Entity function will build the NGSI-LD entities for each line of the csv, magic ! 

## Create the agent 

The entities are created, now you have to create and run the agent by calling your new function.
```python
agent = HttpUploadAgent(sink=sink, process=build_entity)
agent.run()

print(agent.stats)
```
Just like the Sinks, you can create your own agent or use one available in pyngsild [(here)](https://github.com/Orange-OpenSource/pyngsild/tree/main/src/pyngsild)  .

Here we use HttpUploadAgent, very useful for csv conversion. We enter the sink created before and we tell him that we use our build_entity() function to create the entities.

Run, print your agent and congratulations ! You have created your first agent ! 

# Handle the pyngsild bugs
When you have pyngsild bugs you can refer to the common errors right [here](https://ngsildclient.readthedocs.io/en/latest/client.html#mapping-to-ngsi-ld-api-errors).

