# TP3

# TP Extra

## Load Balancing

## Grafana

Pour faire fonctionner Grafana, il faut utiliser Prometheus.    
Nous avons donc rajouter Prometheus dans le application.yml de nos backend.

```yaml
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops,prometheus
```
Aves les dependecies suivantes:

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




