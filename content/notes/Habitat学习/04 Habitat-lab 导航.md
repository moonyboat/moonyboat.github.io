---
title: 04 Habitat-lab 导航
date: 2025-12-14T00:00:00+08:00
draft: false
tags:
  - Habitat
  - Navigation
summary: "  "
---
# Habitat-lab for Navigation


Habitat lab 更侧重于图像化界面，更多负责渲染智能体看到的画面。

不仅是为开发者提供接口，定义全新任务或者自定义智能体，并使用强化学习或者SLAM的方法来训练智能体。

本次将介绍Habitat-lab的核心组件，以及如何添加新任务、新智能体和新传感器，以及如何训练这些智能体。  

因为前面已经讲过的NavMesh的内容，我们可以在habitat-lab中设置一个点导航任务。


# 点导航任务

## Setup PointNav Task

按视频要求，寻找测试用例，但是官网github找了大半圈都没找到，评论区也有和我一样的情况，可能是mp3D的全量数据集？大小有1TB+没有下过全量:(
`"./configs/test/habitat_all_sensors_test.yaml"`文件缺失

下面的代码可以交互式地查看所加载的环境：
```python
    action = None
    obs = env.reset()
    valid_actions = ["TURN_LEFT", "TURN_RIGHT", "MOVE_FORWARD", "STOP"]
    interactive_control = False  # @param {type:"boolean"}
    while action != "STOP":
        display_sample(obs["rgb"])
        print(
            "distance to goal: {:.2f}".format(
                obs["pointgoal_with_gps_compass"][0]
            )
        )
        print(
            "angle to goal (radians): {:.2f}".format(
                obs["pointgoal_with_gps_compass"][1]
            )
        )
        if interactive_control:
            action = input(
                "enter action out of {}:\n".format(", ".join(valid_actions))
            )
            assert (
                action in valid_actions
            ), "invalid action {} entered, choose one amongst " + ",".join(
                valid_actions
            )
        else:
            action = valid_actions.pop()
        obs = env.step(
            {
                "action": action,
            }
        )

    env.close()
```

## RL Training

进行RL训练，需要载入baseline里面的训练设置，如`ppo_pointnav_example.yaml`
```python
    # set random seeds
if __name__ == "__main__":
    seed = "42"  # @param {type:"string"}
    num_updates = "20"  # @param {type:"string"} # 通常很大，演示取的小数字
    config = get_baselines_config( "./habitat_baselines/config/pointnav/ppo_pointnav_example.yaml")
    config.defrost()
    config.TASK_CONFIG.SEED = int(seed)
    config.NUM_UPDATES = int(num_updates) 
    config.LOG_INTERVAL = 1
    config.freeze()

    random.seed(config.TASK_CONFIG.SEED)
    np.random.seed(config.TASK_CONFIG.SEED)
    

```

## Key Concepts

1.[`habitat.sims.habitat_simulator.HabitatSim`](https://github.com/facebookresearch/habitat-lab/blob/master/habitat/sims/habitat_simulator/habitat_simulator.py#L159)
`HabitatSim`类为对上述类的简单封装。
2.[`habitat.core.env.Env`](https://github.com/facebookresearch/habitat-lab/blob/master/habitat/core/env.py)
对智能体、任务和模拟器的抽象类。智能体的训练和评估在此环境中运行。
3.[`habitat.core.env.RLEnv`](https://github.com/facebookresearch/habitat-lab/blob/71d409ab214a7814a9bd9b7e44fd25f57a0443ba/habitat/core/env.py#L278)
通过定义奖励和其他所需组件，扩展了用于强化学习的`Env`类。
4.[`habitat.core.embodied_task.EmbodiedTask`](https://github.com/facebookresearch/habitat-lab/blob/71d409ab214a7814a9bd9b7e44fd25f57a0443ba/habitat/core/embodied_task.py#L242)
定义智能体需要解决的任务。该类包含观测空间、动作空间、测量、模拟器使用的定义。例如：点导航、对象导航。
5.[`habitat.core.dateset.Dataset`](https://github.com/facebookresearch/habitat-lab/blob/4b6da1c4f8eb287cea43e70c50fe1d615a261198/habitat/core/dataset.py#L63)
包含具体任务数据集所需信息的包装，包含定义和与“事件”的交互。
6.[`habitat.core.embodied_task.Measure`](https://github.com/facebookresearch/habitat-lab/blob/master/habitat/core/embodied_task.py#L82)
定义具体任务的指标，例如：[SPL](https://github.com/facebookresearch/habitat-lab/blob/d0db1b55be57abbacc5563dca2ca14654c545552/habitat/tasks/nav/nav.py#L533).
7.[`habitat_baseline`](https://github.com/facebookresearch/habitat-lab/tree/71d409ab214a7814a9bd9b7e44fd25f57a0443ba/habitat_baselines)
RL、SLAM、针对不同具体任务的启发式基线实现。

# 自定义任务

```python
# 导入之前定义的基本设置
config = habitat.get_config(
	config_paths="./configs/test/habitat_all_sensors_test.yaml"
)

# 注册一个任务名称
@registry.register_task(name="TestNav-v0")
class NewNavigationTask(NavigationTask):
    def __init__(self, config, sim, dataset):
        logger.info("Creating a new type of task")
        super().__init__(config=config, sim=sim, dataset=dataset)

	# 定义是否发生碰撞，来决定回合是否结束。
    def _check_episode_is_active(self, *args, **kwargs):
        logger.info(
            "Current agent position: {}".format(self._sim.get_agent_state())
        )
        collision = self._sim.previous_step_collided
        stop_called = not getattr(self, "is_stop_called", False)
        return collision or stop_called

if __name__ == "__main__":
    config.defrost()
    config.TASK.TYPE = "TestNav-v0"
    config.freeze()

    try:
        env.close()
    except NameError:
        pass
    env = habitat.Env(config=config)
        action = None
    env.reset()
    valid_actions = ["TURN_LEFT", "TURN_RIGHT", "MOVE_FORWARD", "STOP"]
    interactive_control = False  # @param {type:"boolean"}
    while env.episode_over is not True:
        display_sample(obs["rgb"])
        if interactive_control:
            action = input(
                "enter action out of {}:\n".format(", ".join(valid_actions)))
            assert ( action in valid_actions), "invalid action {} entered, choose one amongst " + ",".join(valid_actions)
        else:
            action = valid_actions.pop()
        obs = env.step(
            {
                "action": action,
                "action_args": None,
            }
        )
        print("Episode over:", env.episode_over)
    env.close()
```

# 自定义传感器

```python

# 同理，也是需要注册一个传感器名称
@registry.register_sensor(name="agent_position_sensor")
class AgentPositionSensor(habitat.Sensor):
    def __init__(self, sim, config, **kwargs):
        super().__init__(config=config)
        self._sim = sim

    # Defines the name of the sensor in the sensor suite dictionary
    def _get_uuid(self, *args, **kwargs):
        return "agent_position"

    # Defines the type of the sensor
    def _get_sensor_type(self, *args, **kwargs):
        return habitat.SensorTypes.POSITION

    # Defines the size and range of the observations of the sensor
    def _get_observation_space(self, *args, **kwargs):
        return spaces.Box(
            low=np.finfo(np.float32).min,
            high=np.finfo(np.float32).max,
            shape=(3,),
            dtype=np.float32,
        )
    # This is called whenver reset is called or an action is taken
    def get_observation(self, observations, *args, episode, **kwargs):
        return self._sim.get_agent_state().position
        
if __name__ == "__main__":
    config = habitat.get_config(
        config_paths="./configs/test/habitat_all_sensors_test.yaml"
    )

    config.defrost()
    # Now define the config for the sensor
    config.TASK.AGENT_POSITION_SENSOR = habitat.Config()
    # Use the custom name
    config.TASK.AGENT_POSITION_SENSOR.TYPE = "agent_position_sensor"
    # Add the sensor to the list of sensors in use
    config.TASK.SENSORS.append("AGENT_POSITION_SENSOR")
    config.freeze()

    env = habitat.Env(config=config)
    obs = env.reset()
    obs.keys()
    print(obs["agent_position"])
    env.close()
```

# 自定义智能体

```python
# An example agent which can be submitted to habitat-challenge.
# To participate and for more details refer to:
# - https://aihabitat.org/challenge/2020/
# - https://github.com/facebookresearch/habitat-challenge
class ForwardOnlyAgent(habitat.Agent):
    def __init__(self, success_distance, goal_sensor_uuid):
        self.dist_threshold_to_stop = success_distance
        self.goal_sensor_uuid = goal_sensor_uuid

	# 用于智能体失败后的重置操作
    def reset(self):
        pass

    def is_goal_reached(self, observations):
        dist = observations[self.goal_sensor_uuid][0]
        return dist <= self.dist_threshold_to_stop

	# 根据观测结果提供不同的动作
    def act(self, observations):
        if self.is_goal_reached(observations):
            action = HabitatSimActions.STOP
        else:
            action = HabitatSimActions.MOVE_FORWARD
        return {"action": action}
```

# 自定义动作空间

可见Habitat Lab源码，[habitat-lab/examples/new_actions.py](https://github.com/facebookresearch/habitat-lab/blob/main/examples/new_actions.py)

# Sim2Real with Habitat

可见Habitat Lab源码，[habitat/sims/pyrobot/pyrobot.py](https://github.com/facebookresearch/habitat-lab/blob/71d409ab214a7814a9bd9b7e44fd25f57a0443ba/habitat/sims/pyrobot/pyrobot.py).
论文：[Sim2Real Predictivity: Does Evaluation in Simulation Predict Real-World Performance?](https://arxiv.org/abs/1912.06321)
