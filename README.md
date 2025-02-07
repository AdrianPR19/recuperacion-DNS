# recuperacion DNS 

## 1. Establecimiento de los nombres de los servidores
En esta sección, definimos los nombres de host para las máquinas virtuales en Vagrant. Es crucial asegurarnos de que "atlas" y "ceo" tengan sus respectivos nombres bien configurados desde el principio para evitar inconvenientes futuros.

### 1.1 Definir los nombres de host en Vagrant
Para garantizar que cada servidor tenga su nombre asignado correctamente, editamos el archivo `Vagrantfile` e incluimos la siguiente configuración:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end

  # Actualizar sistema operativo
  config.vm.provision "update", type: "shell", inline: <<-SHELL
    apt-get update
  SHELL

  ######################################################################
  # Configuración del servidor 'atlas'
  ######################################################################
  config.vm.define "atlas" do |atlas|
    atlas.vm.hostname = "atlas"  # Asignar nombre de host
    atlas.vm.network "private_network", ip: "192.168.56.10"  # Dirección IP fija

    atlas.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end

  ######################################################################
  # Configuración del servidor 'ceo'
  ######################################################################
  config.vm.define "ceo" do |ceo|
    ceo.vm.hostname = "ceo"  # Asignar nombre de host
    ceo.vm.network "private_network", ip: "192.168.56.11"  # Dirección IP fija

    ceo.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end
end
```

### 1.2 Iniciar las máquinas virtuales
Tras modificar el `Vagrantfile`, ejecutamos el siguiente comando para iniciar las máquinas:

```bash
vagrant up
```



## 2. Asignación de direcciones IP a atlas y ceo
Cada servidor debe contar con una dirección IP fija:

- **Atlas**: `192.168.56.10`
- **Ceo**: `192.168.56.11`

Para comprobar que las direcciones IP se han establecido correctamente, ejecutamos los siguientes comandos en el vagrantfile:

```bash
vagrant ssh atlas
ip a | grep 192.168.56
exit
```

```bash
vagrant ssh ceo
ip a | grep 192.168.56
exit
```

Si las direcciones IP aparecen correctamente en la salida, la configuración está bien aplicada.

## 3. Configuración de la zona DNS "olimpo.test"
En este apartado, configuramos el dominio "olimpo.test" para que **atlas** sea el servidor DNS principal y **ceo** funcione como secundario. Esto permitirá que atlas administre los registros y ceo se mantenga sincronizado con sus cambios.

### 3.1 Definir las zonas en los servidores
#### 3.1.1 Configuración en atlas
Editamos el archivo de configuración de BIND en atlas:

```bash
sudo nano /etc/bind/named.conf.local
```

Añadimos las siguientes líneas:

```conf
zone "olimpo.test" {
    type master;
    file "/etc/bind/db.olimpo";
    allow-transfer { 192.168.56.11; };
    notify yes;
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.olimpo.rev";
    allow-transfer { 192.168.56.11; };
    notify yes;
};
```

Guardamos y cerramos el archivo.

#### 3.1.2 Configuración en ceo
Editamos el mismo archivo en ceo:

```bash
sudo nano /etc/bind/named.conf.local
```

Añadimos la siguiente configuración:

```conf
zone "olimpo.test" {
    type slave;
    file "/var/cache/bind/db.olimpo";
    masters { 192.168.56.10; };
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.olimpo.rev";
    masters { 192.168.56.10; };
};
```

Guardamos y cerramos el archivo.

### 3.2 Crear los archivos de zona en atlas
#### 3.2.1 Zona directa
Editamos el archivo de configuración:

```bash
sudo nano /etc/bind/db.olimpo
```

Y añadimos:

```conf
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        1        ; Serial
        1200     ; Refresh
        900      ; Retry
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

@   IN  NS  atlas.olimpo.test.
atlas   IN  A   192.168.56.10
ceo     IN  A   192.168.56.11
```

El resto del procedimiento sigue el mismo esquema para:

- **Zona inversa** (`/etc/bind/db.olimpo.rev`)
- **Reinicio de BIND** (`sudo systemctl restart bind9`)
- **Verificación de la transferencia de zona** (`ls -l /var/cache/bind/` en ceo)

### 4. Configuración de reenviadores DNS
Para que los servidores reenvíen consultas externas, modificamos `named.conf.options` en **atlas** y **ceo**:

```bash
sudo nano /etc/bind/named.conf.options
```

Dentro de `options {}`, agregamos:

```conf
    forwarders {
        1.1.1.1;
    };
```

Reiniciamos BIND en ambos servidores:

```bash
sudo systemctl restart bind9
```

### 5. Pruebas finales
Para verificar la configuración, ejecutamos:

```bash
dig @192.168.56.11 google.com
```

Si la consulta obtiene una respuesta válida, la configuración es correcta.

---
Este documento cubre la configuración básica de servidores DNS en un entorno Vagrant. Para mayor seguridad y escalabilidad, pueden aplicarse configuraciones adicionales según sea necesario.

