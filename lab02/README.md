### Underlay. OSPF
Цель: Настроить OSPF для Underlay сети

    Настроите OSPF в Underlay сети, для IP связанности между всеми сетевыми устройствами.
    Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
    Убедитесь в наличии IP связанности между устройствами в OSFP домене

###  Сетевая схема
  ![](nettopology.png)  


### План выполнения работ
#### Предусловие
Выполнена коммутация и назначены IP адреса в соответствии с [Заданием 1]{https://github.com/rinatkaa/DataCenterLabs/tree/7e59cc8ceb32a081ec70271e0248e5d9934b8caa/lab01}

####  Настроить процесс маршрутизации OSPF
Используется только Area 0

#### Настроить атрибуты протокола OSPF на интерфейсах

### Настроить BFD

### Выполнить контроль и проверки

### Конфигурации устройств
- Spine коммутаторы:
  - [swsp-dc1-1](configs/swsp-dc1-01-config.txt)
  - [swsp-dc1-2](configs/swsp-dc1-02-config.txt)
- Leaf коммутаторы:
  - [swle-dc1-1](configs/swle-dc1-01-config.txt)
  - [swle-dc1-2](configs/swle-dc1-02-config.txt)
  - [swle-dc1-3](configs/swle-dc1-03-config.txt)
