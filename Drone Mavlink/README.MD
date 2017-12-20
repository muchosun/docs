# Drone Mavlink Module

Данный модуль испльзуется для создания связи между БПЛА и модулем Drone Master Messenger. Модуль использует протокол связи для БПЛА MAVLink, API-интерфейс, совместимый с Ethereum web3.js и фреймворк Robot Operating System. Обратим внимание на ключевые файлы, используещиеся в данном проекте.

## route_mutex.py

Этот файл описывает класс ```Route```, который является абстракцией маршрута дрона, протоколируемого в Ethereum. Класс включает в себя интерфейс взаимодействия с Ethereum и 2 метода.

Файл использует  библиотеку web3, отвечающую за взаимодействие с Ethereum, а также модули json и os:

```python
from web3 import Web3
import json, os
```

Для работы с контрактами Ethereum объявляется json-переменная бинарного интерфейса приложения (ABI). ABI определяет как вызываются функции в контракте и как возвращаются данные транзакций. Сам интерфейс определен в файле контракта *RouteMutex.sol*.

```python
abi = '[{"constant":false,"inputs":[{"name":"_ipfs_hash","type":"string"}],"name":"acquire","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"_ipfs_hash","type":"string"}],"name":"get","outputs":[{"name":"","type":"address"},{"name":"","type":"string"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_ipfs_hash","type":"string"}],"name":"release","outputs":[],"payable":false,"type":"function"}]'
```

Интерфейс состоит из трех методов, которые обеспечивают связь с Ethereum и файлами InterPlanetary File System (IPFS) с помощью хэш-сумм:

* ```acquire``` — метод позволяет занять незанятый маршрут;
* ```get``` — метод чтения;
* ```release``` — метод позволяет освободить занятый маршрут.

Теперь разберем, как определяется сам класс Route и его методы. В первую очередь рассмотрим конструктор класса:

```python
def __init__(self, ipfs_hash):
  ...
```

Конструктор на основании переданного хэша IPFS определяет следующие атрибуты класса:

* ```self.ipfs_hash``` — хэш маршрута, хранящийся в этой файловой системе;

* ```self.contract``` — экземпляр контракта, со всеми его методами и событиями, определенными в интерфейсе ABI: шестнадцатеричный кодированный 20-байтовый адрес контракта подгружается из переменных среды операционной системы;

* ```self.account``` — ключи?

В этом сегменте определяется протокол удаленного вызова процедур:

```python
web3 = Web3(Web3.RPCProvider(host=os.environ['WEB3_HOST']))
manager = Web3.PrivateKeySigningManager(web3._requestManager)
manager.register_private_key(os.environ['WEB3_PRIV_KEY'])
web3.setManager(manager)
```

Теперь рассмотрим два метода класса Route.

1. Метод ```acquire()```

 ```python
 def acquire(self):
    ...
 ```

 Этот метод проводит транзакцию и позволяет занять незанятый маршрут. Метод обращается по интерфейсу к соответствующему методу контракта, используя хэш IPFS.

2. Метод ```release()```

 ```python
 def release(self):
    ...
 ```

 Этот метод проводит транзакцию и освобождает занятый маршрут. Метод обращается по интерфейсу к соответствующему методу контракта, используя хэш IPFS.

##  mission.py

Этот файл представляет собой организацию узла ROS, который отвечает за загрузку миссии для дрона, как в протокол MAVLink, так и в Ethereum.

Файл использует следующий набор библиотек:

```python
import urllib2, rospy, json, os
from mavros_msgs.srv import WaypointPush
from mavros_msgs.msg import Waypoint
from route_mutex import Route
```
Видно, как подключаются заголовочные файлы сообщений и сервисов MAVROS - коммуникационного драйвера протокола связи MAVLink для ROS. В частности подключается тип сообщений ```Waypoint```, кодирующий путевые точки маршрута, а также сервис ```WaypointPush```, отправляющий эти путевые точки. Также подключается определенный в файле *route_mutex.py* класс ```Route```.

Файл содержит в себе следующий метод:

```python
def get_mission(mission_hash):
  ...
```

Этот метод необходим для корректного получения миссии из файловой системы IPFS. Метод обращается к хранящемуся в IPFS файлу миссии по URL, и оттуда подгружаются данные о путевых точках маршрута.

Разберем данный метод детально:

1. Из переменных окружения выгружается переменная ```IPFS_HOST```. Эта переменная и полученный хэш ```mission_hash```  формируют ссылку для дальнейшей выгрузки информации о миссии полета БПЛА. Ссылка помещается в переменную ```mission_url```.

   ```python
   mission_url = 'http://{0}/ipfs/{1}'.format(os.environ['IPFS_HOST'], mission_hash)
   ```
2. Информацию о миссии по URL загружается в переменную ```mission_data```.

    ```python
    mission_data = urllib2.urlopen(mission_url).read()
    ```
3. Далее элементы миссии — путевые точки — выгружаются в отдельную переменную:

   ```python
   mission_items = json.loads(mission_data)['items']
   ```

4. Затем все точки миссии распределяются согласно стандарту MAVLink. Каждая путевая точка обладает множеством полей, которые определяют ее принадлежность к координатной системе, положение (в метрах по x, y, z или в координатах широты, долготы и высоты над уровнем моря), а также служебную информацию протокола связи.

  ```python
for item in mission_items:
      w = Waypoint()
      w.frame = item['frame']
      w.command = item['command']
      w.autocontinue = item['autoContinue']
      w.param1 = item['param1']
      w.param2 = item['param2']
      w.param3 = item['param3']
      w.param4 = item['param4']
      w.x_lat  = item['coordinate'][0]
      w.y_long = item['coordinate'][1]
      w.z_alt  = item['coordinate'][2]
      yield w
  ```

Заметим, что в итоге метод ```get_mission``` является генератором. Учтем это при дальшейшем использовании данного метода.

Итак, мы сформировали метод-генератор по работе с миссией.

Далее инициализируется узел ```mission_loader```, который создает соответствующий топик в ROS.

```python
if __name__ == '__main__':
    rospy.init_node('mission_loader', anonymous=True)
```

Если узел обнаружил среди переменных среды ```os.environ``` хэш миссии ```MISSION_HASH```, то узел сначала пытается связаться с сервисом MAVROS ```mavros/mission/push```:

```python
if 'MISSION_HASH' in os.environ and len(os.environ['MISSION_HASH']) > 0:
        mission = os.environ['MISSION_HASH'].strip()
        rospy.wait_for_service('mavros/mission/push')
        push = rospy.ServiceProxy('mavros/mission/push', WaypointPush)
```

Затем узел пытается загрузить миссию в сеть Ethereum, чтобы занять необходимый маршрут с помощью ранее описанного класса ```Route```. Если все проходит успешно, то организуется соответствующая транзакция и маршрут становится занят этой миссией. Кроме того информация о путевых точках маршрута передается в вызванный сервис MAVROS. При завершении миссии или невозможности загрузить маршрут узел проводит транзакцию об освобождении занятого маршрута.

```python
route = Route(mission)
try:
    route.acquire()
    push(list(get_mission(mission)))
    rospy.loginfo('Route {0} acquired and loaded'.format(mission))
except:
    route.release()
    rospy.logerror('Unable to load route {0}'.format(mission))
```

Если узел не обнаружил среди переменных среды хэш миссии, то он выдает соответствующее предупреждение и загрузка миссии пропускается.

*Примечание:* нужно определить, что именно означает атрибут ```account``` в конструкторе класса. Также нужно описать сегмент с ```web3.setManager```.
