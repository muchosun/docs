## Drone Mavlink Module

Данный модуль испльзуется для создания связи между БПЛА и модулем Drone Master Messenger. Обратим внимание на ключевые файлы используещиеся в данном проекте.

*  mission.py

Данный набор методов использует следующий набор библиотек:

```python
import urllib2, rospy, json, os
from mavros_msgs.srv import WaypointPush
from mavros_msgs.msg import Waypoint
from route_mutex import Route
```

Содержит в себе следующий метод
```python
def get_mission(mission_hash):
  ...
```

Данный метод необходим для корректного получения миссии из системы хранеия данный ipfs.io

Разберем данный метод дитально:

1. Заберем из системной переменной ```IPFS_HOST``` и сформируем ссылку для дальнейшей выгрузки инофрмации по миссии полета БПЛА в переменную ```mission_url``` .

```python
mission_url = 'http://{0}/ipfs/{1}'.format(os.environ['IPFS_HOST'], mission_hash)
```
2. Заберем всю информацию о миссии в переменную ```mission_data```.

 ```python
 mission_data = urllib2.urlopen(mission_url).read()
 ```
3. Далее выгрузим элементы миссии в отдельную переменную:

```python
mission_items = json.loads(mission_data)['items']
```

4. Затем распредилим все точки миссии согласно стандарту Mavlink

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

Заметим что в итоге у нас метод является генератором. Учтем это при дальшейшем использовании данного метода.

И так, мы сформировали метод-генератор по работе с миссией.

Далее иницируем топик в ROS.

```python
if __name__ == '__main__':
    rospy.init_node('mission_loader', anonymous=True)
```

Проверяем в ```os.environ``` наличие элемента ```MISSION_HASH``` и если всетаки данный элемент найден, загружаем элементы миссии непосредственно в БПЛА через метод ```Route```:

```python
if 'MISSION_HASH' in os.environ and len(os.environ['MISSION_HASH']) > 0:
      mission = os.environ['MISSION_HASH'].strip()
      rospy.wait_for_service('mavros/mission/push')
      push = rospy.ServiceProxy('mavros/mission/push', WaypointPush)
      rospy.info('Try to load mission {0}'.format(mission))
      route = Route(mission)
      try:
          route.acquire()
          push(list(get_mission(mission)))
          rospy.loginfo('Route {0} acquired and loaded'.format(mission))
      except:
          route.release()
          rospy.logerror('Unable to load route {0}'.format(mission))
  else:
      rospy.logwarn('No mission specified, skip loading')
```

В противном случае пропускаем данный метод выгрузки мисси.

* route_mutex.py



```python
from web3 import Web3
import json, os
```
