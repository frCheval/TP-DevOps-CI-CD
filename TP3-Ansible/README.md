# TP3

## 1 Document your inventory and base commands

Notre fichier inventories contient les informations qui nous permettent de nous connecter à notre serveur (après avoir correctement setup ansible en local sur notre (TD)).

La variable ```ansible_user``` précise l'utilisateur avec lequel on souhaite effectuer une interaction avec la machine.

La variable  ```ansible_ssh_private_key_file``` précise le chemin local de la clé privée qui permet de se connecter au serveur distant..

Enfin la variable ``` hosts``` égale à ``` francois.cheval.takima.cloud``` correspond à l'adresse du serveur auquel on veut se connecter.

Pour faire un test de connexion à notre serveur, on execute la commande suivante :
```ansible all -i inventories/setup.yml -m ping```

Le serveur va nous répondre :

```
francois.cheval.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```

On peut aussi effectuer d'autres commandes sur le serveur, pour récupérer l'OS du serveur par exemple :

```ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"```

Cela répond :
```
francois.cheval.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "8",
        "ansible_distribution_release": "NA",
        "ansible_distribution_version": "8",
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false
}
```
On peut aussi faire des modifications root sur le serveur en utilisant l'argument ```--become``` qui est l'équivalent d'un sudo.
La commande ci-dessous permet par-exemple de supprimer le serveur apache de notre machine distante:
```ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become```

## 2 Document your playbook

Le playbook permet d'exécuter une suite de commandes ou d'actions sur le serveur distant. Une fois crée et complété dans le bon répertoire, il s'utilise avec la commande suivante :

```ansible-playbook -i inventories/setup.yml playbook.yml```

Dans un premier temps, notre playbook est assez basique puisqu'il exécute des commandes d'installation de docker sur la machine distante et vérifie la bonne installation.

Cependant, nous pouvons aussi créer des roles, dont les actions sont détaillés dans le fichier main.yaml du répertoire tasks du répertoire qui porte le même nom que le role, répertoire situé au même niveau que notre playbook.

La création d'un role se fait avec la commande suivante :

```ansible-galaxy init roles/docker```

Cette commande par-exemple initie un role docker, dans lequel on va mettre les tâches d'installation et de vérifications de Docker sur la machine distante.

Le main.yaml de task du role docker ressemble alors à :
```yaml
---
# tasks file for roles/docker
- name: Clean packages
  command:
    cmd: dnf clean -y packages

- name: Install device-mapper-persistent-data
  dnf:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  dnf:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  dnf:
    name: docker-ce
    state: present

- name: install python3
  dnf:
    name: python3

- name: Pip install
  pip:
    name: docker

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

On peut retirer ces tasks de notre playbook et rajouter une ligne en haut de celui-ci :

```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
```
A noter que si il y a plusieurs roles, leurs têches respectives seront executés dans l'ordre de la liste, c'est-à-dire de haut en bas.

Pour la suite du TP nous avons crée les roles backend, front, proxy, database et network. Notre playbook ressemble donc à :

```yaml
- hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - backend
    - frontend
    - proxy

```

## 3 Document your docker_container tasks configuration.

Dans le role docker, nous configurons la création d'un réseau docker commun à tout nos conteneurs, nommés 'app_network'. Pour cela, nous utilisons la balise docker_network de cette manière :

```yaml
#Creation of network
- name: Create a network
  docker_network:
    name: app_network
```

L'ensemble des autres conteneurs seront associés à ce réseau.

Dans les roles backend, frontend et database, on crée les conteneurs docker pour qu'il existe sur notre serveur distant.

Cela se fait à l'aide de la balise ```docker_container```.

Les infos à rentrer sous cette balise sont globalement les mêmes que nous avons mises dans notre docker-compose. 

Ainsi, on retrouve les variables d'environnement et volumes pour le conteneur BDD :
```yaml
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - ./postgresql/data:/var/lib/postgresql/saved-data
      - ./postgresql/pg-init-scripts:/docker-entrypoint-initdb.d
```
Pour tout les conteneurs, on doit préciser l'image et son tag, pour qu'il puisse les télécharger depuis DockerHub :

```yaml
    name: devops-postgres-tp-1
    image: fcheval/tp3-ansible-postgres-tp:1.0
```

La variable permet aussi de donner un nom à notre conteneur.

Enfin, on peut aussi préciser les ports comme dans cette exemple pour le conteneur proxy :

```
#Launch proxy
- name: Start proxy
  docker_container:
    name: devops-httpd-tp-1
    image: fcheval/tp3-ansible-httpd-tp:1.0
    ports:
      - 80:80
      - 8080:8080
    networks:
      - name: app_network
```

Le tout se relance avec la même commande que tout à l'heure :
```ansible-playbook -i inventories/setup.yml playbook.yml```

Il faut faire preuve d'un peu de patience mais une fois l'ensemble des étapes réalisés, on peut tester la connexion à notre backend en utilisant le nom de notre serveur et le port 8080.

# TP Extra

## Load Balancing

Pour le load balancing, nous avons dû apporter des modifications au docker httpd.   
Pour cela nous avons modifié le fichier httpd.conf en ajoutant les lignes suivantes:

```conf
Listen 80
Listen 8080

...

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://devops-frontend-tp-1:80/
    ProxyPassReverse / http://devops-frontend-tp-1:80/
</VirtualHost>

<VirtualHost *:8080>
    ProxyPreserveHost On
    Header set Access-Control-Allow-Origin "*"
    <Proxy balancer://mycluster>
        BalancerMember "http://devops-backend-tp-1:8080"
        BalancerMember "http://devops-backend-tp-2:8080"
    </Proxy>
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
```

Avec cette configuration httpd de notre serveur Apache, on a définit notre serveur comme un proxy qui accepte les requêtes provenant du conteneur frontend sur le port 80, et les requêtes qui proviennent du backend sur le port 8080.

Pour le port 8080, on a définit un load balancer qui contient l'url de nos deux backends. Ainsi, on précise au proxy de préférer l'un des deux urls en fonction de la charge que recontrent les backends, ou de leurs disponibilités.

De cette manière, notre serveur Apache est le seul conteneur qui doit avoir des ports exposés et il est en charge de toutes les redirections de requêtes. On change donc les ports dans le docker-compose.yml de cette manière :
```yaml
  httpd-tp:
    build: ./httpd
    container_name: devops-httpd-tp-1
    ports:
      - 80:80
      - 8080:8080
    networks:
      - app-network
    depends_on:
      - postgres-tp
```

On a aussi ajouté la création d'un second conteneur pour disposer de deux backends distinct,utilisant la même image et la même BDD. Cette ajout a aussi eu lieu dans le docker-compose :

```yaml
  backend-tp-1:
    build: ./backend
    container_name: devops-backend-tp-1
    networks:
      - app-network
    depends_on:
      - postgres-tp
  
  backend-tp-2:
    build: ./backend
    container_name: devops-backend-tp-2
    networks:
      - app-network
    depends_on:
      - postgres-tp
```


## Grafana

Pour faire fonctionner Grafana, il faut utiliser Prometheus.    
Nous avons donc rajouté Prometheus dans le fichier application.yml de nos backends.

```yaml
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops,prometheus
```
Aves les dependencies suivantes:

```xml
		<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- https://mvnrepository.com/artifact/io.micrometer/micrometer-registry-prometheus -->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
		</dependency>
```

Nous avons ensuite créé un docker Prometheus dans le docker-compose.yml.

```yaml
  prometheus-tp:
    image: prom/prometheus:v2.42.0
    container_name: devops-prometheus-tp-1
    restart: unless-stopped
    volumes:
      - ./prometheus/:/etc/prometheus/
    command:
      - '--config.file=/etc/prometheus/prometheus.yaml'
    ports:
      - 9090:9090
    networks:
      - app-network
    depends_on:
      - backend-tp-1
      - backend-tp-2
```

On voit que le docker Prometheus repose sur une conf prometheus.yml dans le dossier prometheus, on a donc créé ce fichier et on lui donne les deux backends en target.

```yaml
scrape_configs:
  - job_name: 'Spring Boot Application input'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 2s
    static_configs:
      - targets: ['devops-backend-tp-1:8080','devops-backend-tp-2:8080']
        labels:
          application: 'Spring Boot Application'
```

On vérifie que Prometheus fonctionne bien en allant sur http://localhost:9090/ et en vérifiant que les targets sont "UP".

On installe ensuite le docker Grafana.

```yaml
  grafana-tp:
    image: grafana/grafana-oss:8.5.2
    pull_policy: always
    container_name: devops-grafana-tp-1
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - ./grafana:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SERVER_DOMAIN=localhost
    networks:
      - app-network
    depends_on:
      - prometheus-tp
```

On peut ensuite accéder à Grafana à l'adresse http://localhost:3000/ et se connecter avec le login admin et le mot de passe admin.

On rajoute ensuite un datasource Prometheus en allant dans Configuration > Data Sources > Add data source.   
On ajoute l'adresse http://prometheus-tp:9090/ et on valide.    
On peut ensuite créer un dashboard en allant dans Create > Dashboard > Add Query.