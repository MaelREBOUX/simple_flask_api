# simple Flask API

Very simple Python Flask API with nginx + socket configuration

Tutoriel pour créer une API de démonstration déployée sur un serveur linux avec un serveur web nginx.




## Cloner ce projet

On va considérer que l'on travaille dans un répertoire ```/data/projets/```.

```bash
mkdir -p /data/projets/
cd /data/projets/
git clone https://github.com/MaelREBOUX/simple_flask_api.git
cd simple_flask_api/
```

## Configurer un vhost de test dans nginx

On va considérer que l'on travaille sur une machine de test.

Modifier le fichier ```/etc/hosts``` pour avoir un pseudo domaine local.

```bash
sudo nano /etc/hosts
```

Rajouter ```127.0.0.1   hello_api```

Copier ensuite le fichier configuration proposé et activer ce vhost de test.
Considérant que l'on est sous Debian 10 ou Ubuntu 20.04 :

```bash
cp hello_api_nginx_conf /etc/nginx/site-available/hello_api
cd /etc/nginx/site-enable/
ln -s ../site-available/hello_api hello_api
nginx -t
service nginx reload
```

Tester si le serveur web est configuré correctement en allant sur http://hello_api/


## Installer des paquets requis

Il faut installer des paquets permettant de builder les librairies requises :

```bash
sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev libpcre3 libpcre3-dev python3-setuptools
```

## Créer un environnement virtuel Python

```bash
cd /data/projets/simple_flask_api/
python3 -m venv venv
. venv/bin/activate
pip install flask uwsgi
```

Tester

```bash
python3 hello.py
```

Vérifier que l'on obtient bien un "Hello There" sur la [page http://hello_api:5000/](http://hello_api/hello/about/)

Faire `ctrl + C` pour arrêter.


## Tester uWSGI

On teste maintenant si uWSGI et le socket fonctionnent bien en utilisant le fichier de configuration ```hello.ini``` :

```bash
uwsgi --ini hello.ini
```

Faire ```ll /tmp/hello*``` pour voir si le socket a bien été créé par www-data.

Tester si on obtient bien toujours un "Hello There" sur la page [http://hello_api/hello/](http://hello_api/hello/about/)

À ce stade tout est fonctionnel mais il faut maintenant créer un daemon pour ne pas avoir une commande uwsgi dans le shell.

Sortir du mode venv en tapant : ```deactivate```.



## Configurer un socket spécifique permanent sur la machine

Il s'agit ici de configurer un socket sépcifique qui sera démarré automatiquement au boot.

```navigateur --> nginx http://hello_api/hello/ :80 --> socket unix localhost --> hellopy```

En décodé : la route ```/hello/``` demandée à nginx sera routée vers un serveur Python local abrité derrière un socket pour servir le script python qui répondra.

On va donc créer un démon sous system.d.

Créer un fichier "unit" :

```bash
sudo nano /etc/systemd/system/api_hello.service
```

avec les directives ci-dessous :

```
[Unit]
Description=uWSGI instance to serve Hello API
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/data/projets/simple_flask_api
Environment="PATH=/data/projets/simple_flask_api//venv/bin"
ExecStart=/data/projets/simple_flask_api/venv/bin/uwsgi --ini hello.ini

[Install]
WantedBy=multi-user.target
```

On n'oublie pas de mettre les permissions au serveur web sur les fichiers :

```bash
chown -R www-data:www-data /data/projets/simple_flask_api/
```

Et on crée le répertoire pour les logs (en option) :

```bash
mkdir -p /var/log/uwsgi/
```

Lancer et tester le service :

```bash
# enable
sudo systemctl enable api_hello
# start
sudo systemctl start api_hello
# check
sudo systemctl status api_hello
```

Faire ```ll /tmp/hello*``` pour voir si le socket a bien été créé par www-data.

Tester si on obtient bien toujours un "Hello There" sur la page [http://hello_api/hello/](http://hello_api/hello/about/)

On peut aussi tester [http://hello_api/hello/about/](http://hello_api/hello/about/)


Si on veut supprimer le service :

```bash
systemctl stop api_hello
systemctl disable api_hello
rm /etc/systemd/system/api_hello.service
systemctl daemon-reload
systemctl reset-failed
```

## Sources

https://flask.palletsprojects.com/en/1.1.x/deploying/uwsgi/

https://www.vultr.com/docs/deploy-a-flask-website-on-nginx-with-uwsgi

https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-20-04

https://medium.com/@ksashok/using-nginx-for-production-ready-flask-app-with-uwsgi-9da95d8ac0f9

https://uwsgi-docs.readthedocs.io/en/latest/Nginx.html

https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html#putting-behind-a-full-webserver

https://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html#automatically-starting-uwsgi-on-boot
