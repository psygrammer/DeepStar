
# Main Reference

- [How To: PySC2](https://github.com/skjb/pysc2-tutorial)
- 목표: 자신의 SC2 봇 만들기
- 일차적으로 테란 중심으로 기술. (이후 저그나 프로토스로 확장 예정)

# 목차
## (Building a Basic Agent)[https://itnext.io/build-a-zerg-bot-with-pysc2-2-0-295375d2f58e]
## Building a Simple RL Agent
## Add Smart Attacking to Your Agent
## Building a Sparse Reward Agent
## Refine Your Sparse PySC2 Agent

# Building a Zerg Bot with PySC2 2.0

- [Link](https://chatbotslife.com/building-a-basic-pysc2-agent-b109cde1477c) 은 PySC2 2.0 이전 버젼으로 가동되던 것 --> [Link](https://itnext.io/build-a-zerg-bot-with-pysc2-2-0-295375d2f58e)




```python
# 이 부분은 발제자 import 문제로 추가되어 있는 것임. 다른 분들은 실행할 필요 없음
import sys
sys.path.append("/Users/j/Documents/seminar/2019/DeepStar/venv/lib/python3.7/site-packages")
```

## Creating the Basic Agent
### Overall Process

- Importing Essential Modules --> Add the Run Code --> Select a Drone --> Build a Spawning Pool --> Build Zerglings --> Spawn More Overloads --> Attack!
- 주의사항: 아래 코드는 튜토리얼 특성상 부분 실행보다는 통으로 실행하길 권장

### Importing Essential Modules


```python
from pysc2.agents import base_agent
from pysc2.env import sc2_env
from pysc2.lib import actions, features
from absl import app
```

### Creating Agent


```python
class ZergAgent(base_agent.BaseAgent):
  def step(self, obs):
    super(ZergAgent, self).step(obs)
    
    return actions.FUNCTIONS.no_op()
```

- The ```step``` method is the core part of our agent, ***it’s where all of our decision making takes place.*** ***At the end of every step you must return an action,*** in this case the action is to do nothing. We will add some more actions soon.
- If you followed my previous tutorials you will notice that the format for the action has changed, previously the last line would have been:

### Add the Run Code


```python
def main(unused_argv):
    agent = ZergAgent()
    try:
        while True:
            with sc2_env.SC2Env(
                map_name="Automaton", # 맵을 오토메이톤으로 세팅
                players=[sc2_env.Agent(sc2_env.Race.zerg), # 저그종족.
                       # 다른 선택 가능 종족은 protoss, terran, random 이 있음
                    sc2_env.Bot(sc2_env.Race.random,
                        sc2_env.Difficulty.very_easy)],
                        # 상대를 Bot으로 할 것이며, 종족은 random, 난이도는 매우 쉬움
                        # 여기에 다른 agent를 설정할 수 있음
                    agent_interface_format=features.AgentInterfaceFormat(
                        feature_dimensions=features.Dimensions(screen=84, minimap=64)),
                        step_mul=16,
                        # 액션 설정. 8 -> 300APM, 16 --> 150APM 
                        game_steps_per_episode=0,
                        # 디폴트는 30분인데, 여기에서는 endless로 설정
                        visualize=True) as env:
                        # 스크린과 미니맵 해상도 설정
                        # PySC2 2.0에서는 RGB 레이어를 추가할 수 있음 
                agent.setup(env.observation_spec(), env.action_spec())

                timesteps = env.reset()
                agent.reset()

                while True:
                    step_actions = [agent.step(timesteps[0])]
                    if timesteps[0].last():
                        break
                    timesteps = env.step(step_actions)
      
    except KeyboardInterrupt:
        pass
  
    if __name__ == "__main__":
        app.run(main)
```

- 실행전에 ```[Startraft2 root]/Maps/Ladder2019Season1/``` 폴더에 ```AutomatonLE.SC2Map``` 맵을 복사해야 함. 
- ```AutomatonLE.SC2Map``` 파일은 본 문서가 있는 폴더에 업로드
    - 예제에서는 Abyssal Reef LE 맵을 사용했지만 다른 맵을 사용해도 무방함. 여기에서는 맵 파일을 구할 수 있었던 오토메이턴을 사용
- ```python zerg_agent.py``` 실행
- zerg_agent.py 파일은 본 문서가 있는 폴더에 있음. 

### Select a Drone
- 최초 공격 유닛인 저글링을 생산하기 전 단계로 필요 건물인 스포닝 풀(산란못)이 필요함. 저그의 경우에는 건물을 드론(일벌레)이 변태하는 방식으로 짓기 때문에 이를 위해서 필요한 최초 행동은 드론을 선택하는 것임


```python
from pysc2.lib import actions, features, units
import random
```

- feature unit 사용가능하게 만들기


```python
        agent_interface_format=features.AgentInterfaceFormat(
            feature_dimensions=features.Dimensions(screen=84, minimap=64),
            use_feature_units=True),
```

- step() 안에 화면내 모든 드론 (일벌레)의 리스트를 포착


```python
def step(self, obs):
    super(ZergAgent, self).step(obs)
  
    drones = [unit for unit in obs.observation.feature_units
              if unit.unit_type == units.Zerg.Drone]
    if len(drones) > 0:
        drone = random.choice(drones)
        return actions.FUNCTIONS.select_point("select_all_type", (drone.x, drone.y))
```

- 일벌레 중에 아무거나 하나 집기. 이것은 게임 내에서 <kbd>CTRL</kbd> + 클릭과 동일함

### Build a Spawning Pool


```python
  def unit_type_is_selected(self, obs, unit_type):
    if (len(obs.observation.single_select) > 0 and
        obs.observation.single_select[0].unit_type == unit_type):
      return True
    
    if (len(obs.observation.multi_select) > 0 and
        obs.observation.multi_select[0].unit_type == unit_type):
      return True
    
    return False
```

- ```unit_type_is_selected()```는 리스트 중 첫번째 유닛이 원하는 타입인지 체크함
- ```step()``` 에서 사용할 것임


```python
  def step(self, obs):
    super(ZergAgent, self).step(obs)
    
    if self.unit_type_is_selected(obs, units.Zerg.Drone):
        if (actions.FUNCTIONS.Build_SpawningPool_screen.id in 
          obs.observation.available_actions):
                
            x = random.randint(0, 83)
            y = random.randint(0, 83)
        
        return actions.FUNCTIONS.Build_SpawningPool_screen("now", (x, y))
```

- 크립위이길 바라며 랜덤 포인트를 선정 --> 스포닝 풀을 짓기. 
- 몇 가지 상황이 있을 수 있음 (자원부족, 크립 위가 아님, 다른 건물과 겹침, 지상 유닛이 그 위에 있음 등)
- 또한 스포닝 풀이 여러개 건설 가능한 관계로 무척 많은 스포닝 풀이 건설될 가능성도 있음. 이를 피하기 위해서 아래 코드를 사용


```python
  def get_units_by_type(self, obs, unit_type):
    return [unit for unit in obs.observation.feature_units
            if unit.unit_type == unit_type]
```

- 그러한 결과는 아래와 같음
```python
    spawning_pools = self.get_units_by_type(obs, units.Zerg.SpawningPool)
    if len(spawning_pools) == 0:
      if self.unit_type_is_selected(obs, units.Zerg.Drone):
        if (actions.FUNCTIONS.Build_SpawningPool_screen.id in 
            obs.observation.available_actions):
          x = random.randint(0, 83)
          y = random.randint(0, 83)
          
          return actions.FUNCTIONS.Build_SpawningPool_screen("now", (x, y))
        
      drones = self.get_units_by_type(obs, units.Zerg.Drone)
      if len(drones) > 0:
        drone = random.choice(drones)

        return actions.FUNCTIONS.select_point("select_all_type", (drone.x,
                                                                  drone.y))
```
- 전체 코드는 아래와 같음 (```> python zerg_agent_step4.py```)


```python
from pysc2.agents import base_agent
from pysc2.env import sc2_env
from pysc2.lib import actions, features, units
from absl import app
import random

class ZergAgent(base_agent.BaseAgent):
  def unit_type_is_selected(self, obs, unit_type):
    if (len(obs.observation.single_select) > 0 and
        obs.observation.single_select[0].unit_type == unit_type):
      return True
    
    if (len(obs.observation.multi_select) > 0 and
        obs.observation.multi_select[0].unit_type == unit_type):
      return True
    
    return False

  def get_units_by_type(self, obs, unit_type):
    return [unit for unit in obs.observation.feature_units
            if unit.unit_type == unit_type]
  
  def step(self, obs):
    super(ZergAgent, self).step(obs)
    
    spawning_pools = self.get_units_by_type(obs, units.Zerg.SpawningPool)
    if len(spawning_pools) == 0:
      if self.unit_type_is_selected(obs, units.Zerg.Drone):
        if (actions.FUNCTIONS.Build_SpawningPool_screen.id in 
            obs.observation.available_actions):
          x = random.randint(0, 83)
          y = random.randint(0, 83)
          
          return actions.FUNCTIONS.Build_SpawningPool_screen("now", (x, y))
    
      drones = self.get_units_by_type(obs, units.Zerg.Drone)
      if len(drones) > 0:
        drone = random.choice(drones)

        return actions.FUNCTIONS.select_point("select_all_type", (drone.x,
                                                                  drone.y))
    
    return actions.FUNCTIONS.no_op()

def main(unused_argv):
  agent = ZergAgent()
  try:
    while True:
      with sc2_env.SC2Env(
          map_name="AbyssalReef",
          players=[sc2_env.Agent(sc2_env.Race.zerg),
                   sc2_env.Bot(sc2_env.Race.random,
                               sc2_env.Difficulty.very_easy)],
          agent_interface_format=features.AgentInterfaceFormat(
              feature_dimensions=features.Dimensions(screen=84, minimap=64),
              use_feature_units=True),
          step_mul=16,
          game_steps_per_episode=0,
          visualize=True) as env:
          
        agent.setup(env.observation_spec(), env.action_spec())
        
        timesteps = env.reset()
        agent.reset()
        
        while True:
          step_actions = [agent.step(timesteps[0])]
          if timesteps[0].last():
            break
          timesteps = env.step(step_actions)
      
  except KeyboardInterrupt:
    pass
  
if __name__ == "__main__":
  app.run(main)
```

### Build Zerglings

- 스포닝풀 (산란못)이 완성되면 저글링 만들 준비가 됨. 
- 저그유닛은 해처리(부화장)에서 시간마다 생성되는 라바(애벌래)를 선택하여 원하는 가능한 유닛을 변태하는 과정으로 생성이 진행됨
- 따라서 이를 위해 필요한 첫 행동은 모든 라바를 선택하는 것임
```python

    larvae = self.get_units_by_type(obs, units.Zerg.Larva)
    if len(larvae) > 0:
      larva = random.choice(larvae)
      
      return actions.FUNCTIONS.select_point("select_all_type",
                                            (larva.x, larva.y))```
- 저글링을 만드는 코드는 다음과 같이 설정
```python

    if self.unit_type_is_selected(obs, units.Zerg.Larva):
      if (actions.FUNCTIONS.Train_Zergling_quick.id in 
          obs.observation.available_actions):
        return actions.FUNCTIONS.Train_Zergling_quick("now")
```

### Spawn More Overloads
- 인구수가 부족할 경우 유닛 생성이 안되므로 저그의 인구수 증가를 위해 오버로드 (대군주)를 일부 변태시켜야 함
```python

      free_supply = (obs.observation.player.food_cap -
                     obs.observation.player.food_used)
      if free_supply == 0:
        if (actions.FUNCTIONS.Train_Overlord_quick.id in
            obs.observation.available_actions):
          return actions.FUNCTIONS.Train_Overlord_quick("now")
```
- 행동이 가능한지 체크하는 함수
```python

  def can_do(self, obs, action):
    return action in obs.observation.available_actions
```
- 이 함수를 앞에서 사용했던 액션에 모두 적용 (불가능할 경우 하지 않게 하기 위해서임). 예를 들면:
```python
if self.can_do(obs, actions.FUNCTIONS.Build_SpawningPool_screen.id):
```
여기까지의 전체 코드는 다음과 같음. ```> python zerg_agent_step6.py```


```python
from pysc2.agents import base_agent
from pysc2.env import sc2_env
from pysc2.lib import actions, features, units
from absl import app
import random

class ZergAgent(base_agent.BaseAgent):
  def unit_type_is_selected(self, obs, unit_type):
    if (len(obs.observation.single_select) > 0 and
        obs.observation.single_select[0].unit_type == unit_type):
      return True
    
    if (len(obs.observation.multi_select) > 0 and
        obs.observation.multi_select[0].unit_type == unit_type):
      return True
    
    return False

  def get_units_by_type(self, obs, unit_type):
    return [unit for unit in obs.observation.feature_units
            if unit.unit_type == unit_type]
  
  def can_do(self, obs, action):
    return action in obs.observation.available_actions

  def step(self, obs):
    super(ZergAgent, self).step(obs)
    
    spawning_pools = self.get_units_by_type(obs, units.Zerg.SpawningPool)
    if len(spawning_pools) == 0:
      if self.unit_type_is_selected(obs, units.Zerg.Drone):
        if self.can_do(obs, actions.FUNCTIONS.Build_SpawningPool_screen.id):
          x = random.randint(0, 83)
          y = random.randint(0, 83)
          
          return actions.FUNCTIONS.Build_SpawningPool_screen("now", (x, y))
    
      drones = self.get_units_by_type(obs, units.Zerg.Drone)
      if len(drones) > 0:
        drone = random.choice(drones)

        return actions.FUNCTIONS.select_point("select_all_type", (drone.x,
                                                                  drone.y))
    
    if self.unit_type_is_selected(obs, units.Zerg.Larva):
      free_supply = (obs.observation.player.food_cap -
                     obs.observation.player.food_used)
      if free_supply == 0:
        if self.can_do(obs, actions.FUNCTIONS.Train_Overlord_quick.id):
          return actions.FUNCTIONS.Train_Overlord_quick("now")

      if self.can_do(obs, actions.FUNCTIONS.Train_Zergling_quick.id):
        return actions.FUNCTIONS.Train_Zergling_quick("now")
    
    larvae = self.get_units_by_type(oba, units.Zerg.Larva)
    if len(larvae) > 0:
      larva = random.choice(larvae)
      
      return actions.FUNCTIONS.select_point("select_all_type", (larva.x,
                                                                larva.y))
    
    return actions.FUNCTIONS.no_op()

def main(unused_argv):
  agent = ZergAgent()
  try:
    while True:
      with sc2_env.SC2Env(
          map_name="AbyssalReef",
          players=[sc2_env.Agent(sc2_env.Race.zerg),
                   sc2_env.Bot(sc2_env.Race.random,
                               sc2_env.Difficulty.very_easy)],
          agent_interface_format=features.AgentInterfaceFormat(
              feature_dimensions=features.Dimensions(screen=84, minimap=64),
              use_feature_units=True),
          step_mul=16,
          game_steps_per_episode=0,
          visualize=True) as env:
          
        agent.setup(env.observation_spec(), env.action_spec())
        
        timesteps = env.reset()
        agent.reset()
        
        while True:
          step_actions = [agent.step(timesteps[0])]
          if timesteps[0].last():
            break
          timesteps = env.step(step_actions)
      
  except KeyboardInterrupt:
    pass
  
if __name__ == "__main__":
  app.run(main)
```

### Attack

- 공격을 가기 전에 공격 대상 위치를 알아야 함. 
- 여기에서는 문제를 단순화하기 위해 좌상단 우하단에만 기지가 생성(어비셜 리프 기준. 오토메이톤은 다를 수 있음) 되므로 자기가 어느 위치인지만 파악하여 공격가는 것으로 함. 
- ```__init()__``` 생성하여 관련 변수 (```attack_coordinates```) 설정

```python
  def __init__(self):
    super(ZergAgent, self).__init__()
    
    self.attack_coordinates = None
```

- ```step()``` 를 수정

```python
  def step(self, obs):
    super(ZergAgent, self).step(obs)
    
    if obs.first():  # 첫 스텝이라면 ==> 우리 유닛들의 중심점 좌표를 구함
      player_y, player_x = (obs.observation.feature_minimap.player_relative ==
                            features.PlayerRelative.SELF).nonzero()
      xmean = player_x.mean()
      ymean = player_y.mean()
      
      if xmean <= 31 and ymean <= 31:
        self.attack_coordinates = (49, 49)
      else:
        self.attack_coordinates = (12, 16)
```

- 공격 지시

```python

    zerglings = self.get_units_by_type(obs, units.Zerg.Zergling)
    if len(zerglings) > 0:
      if self.can_do(obs, actions.FUNCTIONS.select_army.id):
        return actions.FUNCTIONS.select_army("select")
```

- 하지만 이렇게 하면 뽑는 대로 공격감. 따라서 일정 수 이상 모인 뒤에 갈 필요가 있음

```python

    if len(zerglings) >= 10:
```

이 파트까지의 내용은 같은 폴더의 zerg_agent_step7.py 에 있음

```>python zerg_agetn_step7.py```



```python

```
