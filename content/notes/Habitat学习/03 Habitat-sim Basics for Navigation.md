---
title: 03 Habitat-sim 基础
date: 2025-12-06
draft: false
tags:
  - Habitat
  - Navigation
summary: "  "
---
# Habitat-sim Basics for Navigation

Habitat平台依赖于许多关键抽象类，这些抽象类对可以在三维室内模拟环境中执行的智能体和任务进行建模。
- **Agent**：一个具有一套传感器的虚拟智能体。能够观察环境，并能够采取行动改变代理或环境状态。
- **Sensor**：传感器，与特定代理相关联，能够以指定的频率从环境中返回观测数据。
- **Scene**：场景，一个包含场景网格、对象、代理和传感器的3D环境。
- **SceneGraph**：场景的分层表示，将环境组织成区域和对象。可以通过编程方式操纵。所有场景组件都显示在SceneGraph上。
- **Simulator**：模拟器后端的实例化。给定一组已配置的Agent和SceneGraphs的操作，可以更新Agent和ScenoGraphs的状态，并为Agent拥有的所有活动传感器提供观测。
本教程涵盖了使用Habitat sim进行导航任务的基础知识，包括：
- 模拟器、传感器和智能体的配置。
- 采取行动并检索观察结果
- NavMesh上的寻路和导航

# Hello Habitat

导航栖息地模拟器由**3**个重要概念组成：可配置的Agent，多个传感器和场景。场景为通用的3D数据集处理（例如，Matterport、Gibson和Replica数据集）。在下面的示例中，我们演示了如何使用传感器设置1个Agent，将其放置在场景中，指示其导航并收集观察结果。（# hello_habitat.py）

配置模拟器后端，重点注意两个类
`sim_cfg = habitat_sim.SimulatorConfiguration`
用于导入场景路径scene_file

`agent_cfg = habitat_sim.agent.AgentConfiguration()`
用于配置智能体，可以导入智能体用到的传感器
`agent_cfg.sensor_specifications = [rgb_sensor_spec]`

配置RGB相机
`rgb_sensor_spec = habitat_sim.CameraSensorSpec()`

返回总设置`return habitat_sim.Configuration(sim_cfg, [agent_cfg])`

```python
def make_simple_cfg(settings):
	# 1. 配置模拟器后端
	sim_cfg = habitat_sim.SimulatorConfiguration()
	# 根目录: /home/.../habitat_ws/habitat2-data
	# 场景路径: scene_datasets/habitat-test-scenes/apartment_1.glb
	scene_file = os.path.join(settings["data_path"], "scene_datasets/habitat-test-scenes/apartment_1.glb")
	
	if not os.path.exists(scene_file):
		raise FileNotFoundError(f"未找到场景文件，请检查路径: {scene_file}")
	
	sim_cfg.scene_id = scene_file
	
	# 2. 配置 Agent (机器人)
	agent_cfg = habitat_sim.agent.AgentConfiguration()
	# 设置传感器 (RGB 相机)
	rgb_sensor_spec = habitat_sim.CameraSensorSpec()
	rgb_sensor_spec.uuid = "color_sensor"
	rgb_sensor_spec.sensor_type = habitat_sim.SensorType.COLOR
	rgb_sensor_spec.resolution = [settings["height"], settings["width"]]
	rgb_sensor_spec.position = [0.0, 1.5, 0.0]
	agent_cfg.sensor_specifications = [rgb_sensor_spec]

	return habitat_sim.Configuration(sim_cfg, [agent_cfg])
```


# NavMesh

在前面的部分中，我们认为导航约束和碰撞响应是理所当然的。默认情况下，这在我们演示的离散Habitat sim动作空间中启用。然而，当直接修改智能体状态时，智能体在采取行动时既不会感觉到障碍物，也不会感觉到场景的边界。我们需要引入一种轻便快捷的机制来执行这些约束。本节将提供有关该方法的更多详细信息。

Habitat sim通过与\[Recast Navigation | Detour]集成提供寻路和导航，本教程部分演示了为静态场景加载、重新计算和保存NavMesh，以及将其明确用于离散和连续导航任务。

导航网格（NavMesh）是一组二维凸多边形（即多边形网格），它们定义了具有特定形状的智能体可以遍历环境的哪些区域。智能体可以在这些区域内自由导航，不受环境中的物体、墙壁、间隙、悬垂物或其他障碍物的阻碍。相邻的多边形在图形中相互连接，使高效的寻路算法能够绘制NavMesh上点之间的路线。

![](images/截图%202025-12-06%2016-36-25.png)

使用导航性的NavMesh近似值，智能体被表示为与重力方向对齐的刚性圆柱体。然后，通过对静态场景进行体素化，并在实体体素的顶面上生成多边形来计算NavMesh，圆柱体将位于该顶面上，没有交叉或悬垂，并遵守配置的约束，如最大爬坡坡度和台阶高度。

## 可视化NavMesh：自顶向下地图

PathFinder API可以轻松地在场景中生成导航性的自上而下映射。由于NavMesh是一个3D网格，并且场景可以垂直具有多个楼层或标高，因此我们需要在特定的世界高度（y坐标）对NavMesh进行切片。然后，通过以可配置的分辨率（meters_per_pixel）对NavMesh进行采样，并留出0.5米的垂直松弛，生成地图。


## PathFinder API
 PathFinder API：Habitat-Sim 使用 `PathFinder` 类来管理 NavMesh。`PathFinder`继承自`habitat_sim.Simulator`类，即：
 ```python
 sim = habitat_sim.Simulator(cfg)
 pathfinder = sim.pathfinder 
 ```
它提供了以下关键能力：

### 基础管理与属性
- **.load_nav_mesh(path)**: 加载指定路径的 NavMesh 文件（`.navmesh`），成功返回 `True`。
- **.is_loaded**: （属性）布尔值，检查 NavMesh 是否已经成功加载。
- **.get_bounds()**: 获取 NavMesh 的轴对齐边界框`（AABB）`，返回 `(min, max)` 坐标，用于确定场景范围。
- **.navigable_area**: （属性）查看当前加载的 NavMesh 的总可导航区域面积，单位是平方米。
- **.seed(seed)**: 设置随机数种子，用于确保 `get_random_navigable_point` 等结果的可复现性。

### 点位查询与修正
- **.is_navigable(point, max_y_delta)**: 检查给定的 3D 坐标点是否可导航（即是否在 NavMesh 上）。`max_y_delta` 允许一定的高度误差。
- **.get_random_navigable_point()**: 在 NavMesh 上随机返回一个均匀采样的可导航点（常用于随机生成 Agent 初始位置）。
- **.snap_point(point, island_index=-1)**: **非常重要**。将一个可能悬空或稍微偏移的 3D 点“吸附”到最近的 NavMesh 表面上。如果点距离 NavMesh 太远，可能返回 `NaN`。
- **.island_radius(point)**: 返回给定点所在的连通区域（Island）的最小包围圆半径。常用于判断一个点是在宽阔的地板上（半径大），还是在孤立的桌子上（半径小）。

### 路径规划与障碍物
- **.find_path(ShortestPath_object)**: 核心寻路函数。需要先创建一个 `habitat_sim.ShortestPath` 对象并设置起点终点，传入此函数后计算最短路径。
- **.distance_to_closest_obstacle(point, max_search_radius)**: 计算给定点到最近障碍物（NavMesh 边界）的距离。
- **.closest_obstacle_surface_point(point, max_search_radius)**: 返回最近障碍物表面的详细信息（位置、法线、距离）。

### 可视化
- **.get_topdown_view(meters_per_pixel, height)**: 生成指定高度切片的 2D 顶视图（Top-down Map），返回一个布尔类型的 2D 数组（True 代表可通行，False 代表障碍）。

## 运行时重新计算NavMesh

在运行时计算NavMesh时，配置选项可用于根据预期用例定制结果。这些设置包括（所有数量均以世界单位表示）：
- **体素化参数**：
	*以更高的计算成本为代价，减少这些值以获得更好的准确性*
	注意：大多数连续参数都转换为单元格尺寸的倍数，因此这些值应该是兼容的，以获得最佳精度。
	- **cell_size** -  xz平面体素尺寸。\[限制：>=0]
	- **cell_height** -  y轴体素维度。\[限制：>=0]
- **智能体设置参数**：
	- **agent_height**-智能体的高度。用于清除有障碍物的通航细胞。
	- **agent_radius**-智能体的半径。用作侵蚀/缩小计算高度场的距离。\[限制：>=0]
	- **agent_max_climp**-仍可通行的最大飞行高度。\[限制：>=0]
	- **agent_max_slope**-可航行的最大坡度。\[限制：0<=值<85] \[单位：度]
- **导航区域过滤选项**（默认激活）：
	- **filter_low_hanging_obstrucles**-如果跨度上方的净空小于指定高度，则将可通航跨度标记为不可通航。
	- **filter_ledge_spans**-将作为ledge结构标记为不可导航。该过滤器减少了保守体素化高估的影响，因此生成的网格在ledge上方出现悬空区域。
	- **filter_walkalble_low_height_spans**-如果跨度上方的净空小于指定高度，则将可导航跨度标记为不可导航。允许形成通航区域，这些区域将流过低矮的物体，如路缘石，并向上流过楼梯等结构。
- **详细网格生成参数**：
	- **region_min_size**-形成封闭区域所允许的最小单元数。
	- **region_merge_size**-合并区域的大小限制，如果可能的话，任何单元格数较小的二维区域都将与较大的区域合并。\[限制：>=0]
	- **edge_max_len**-沿网格边界的轮廓边的最大允许长度。将根据需要插入额外的顶点，以将轮廓边保持在此长度以下。零值有效地禁用了此功能。\[限制：>=0]\[/cell_size]
	- **edge_max_rerror**-简化轮廓的边界边缘应偏离原始原始轮廓的最大距离。\[限制：>=0]
	- **verts_per_poly**-在轮廓到多边形转换过程中生成的多边形允许的最大顶点数。\[限制：>=3]
	- **detail_sample_dist**-设置生成细节网格时使用的采样距离。（仅用于高度细节）\[限制：0或>=0.9]\[x cell_size]
	- **detail_sample_max_error**-细节网格曲面应偏离高度场数据的最大距离。（仅用于高度）\[限制：>=0]\[x单元格高度]

## 在NavMesh上操作

智能体在NavMesh上执行随机动作，连续和离散动作空间都可以。并且对滑动与非滑动场景进行了比较。
### 什么是滑动？

大多数游戏引擎允许代理在指挥与环境碰撞的动作时沿着障碍物滑动。虽然这在游戏中是一种合理的行为，但它并不能准确反映机器人代理与环境之间碰撞的结果。

我们注意到，**允许滑动**会使训练更容易，并导致更高的模拟性能，但会损害训练策略的Sim2Real。有关此主题的更详细说明，请参阅论文：
["Are We Making Real Progress in Simulated Environments? Measuring the Sim2Real Gap in Embodied Visual Navigation"](https://arxiv.org/abs/1912.06321).

Habitat-sim中，使用`sim.config.sim_cfg.allow_sliding = True`来设置滑动生效。

```python
# convert 3d points to 2d topdown coordinates
def convert_points_to_topdown(pathfinder, points, meters_per_pixel):
    points_topdown = []
    bounds = pathfinder.get_bounds()
    for point in points:
        # convert 3D x,z to topdown x,y
        px = (point[0] - bounds[0][0]) / meters_per_pixel
        py = (point[2] - bounds[0][2]) / meters_per_pixel
        points_topdown.append(np.array([px, py]))
    return points_topdown


# display a topdown map with matplotlib
def display_map(topdown_map, key_points=None):
    plt.figure(figsize=(12, 8))
    ax = plt.subplot(1, 1, 1)
    ax.axis("off")
    plt.imshow(topdown_map)
    # plot points on map
    if key_points is not None:
        for point in key_points:
            plt.plot(point[0], point[1], marker="o", markersize=10, alpha=0.8)
    plt.show(block=False)


# @markdown ###Configure Example Parameters:
# @markdown Configure the map resolution:
meters_per_pixel = 0.1  # @param {type:"slider", min:0.01, max:1.0, step:0.01}
# @markdown ---
# @markdown Customize the map slice height (global y coordinate):
custom_height = False  # @param {type:"boolean"}
height = 1  # @param {type:"slider", min:-10, max:10, step:0.1}
# @markdown If not using custom height, default to scene lower limit.
# @markdown (Cell output provides scene height range from bounding box for reference.)

print("The NavMesh bounds are: " + str(sim.pathfinder.get_bounds()))
if not custom_height:
    # get bounding box minumum elevation for automatic height
    height = sim.pathfinder.get_bounds()[0][1]

if not sim.pathfinder.is_loaded:
    print("Pathfinder not initialized, aborting.")
else:
    # @markdown You can get the topdown map directly from the Habitat-sim API with *PathFinder.get_topdown_view*.
    # This map is a 2D boolean array
    sim_topdown_map = sim.pathfinder.get_topdown_view(meters_per_pixel, height)

    if display:
        # @markdown Alternatively, you can process the map using the Habitat-Lab [maps module](https://github.com/facebookresearch/habitat-api/blob/master/habitat/utils/visualizations/maps.py)
        hablab_topdown_map = maps.get_topdown_map(
            sim.pathfinder, height, meters_per_pixel=meters_per_pixel
        )
        recolor_map = np.array(
            [[255, 255, 255], [128, 128, 128], [0, 0, 0]], dtype=np.uint8
        )
        hablab_topdown_map = recolor_map[hablab_topdown_map]
        print("Displaying the raw map from get_topdown_view:")
        display_map(sim_topdown_map)
        print("Displaying the map from the Habitat-Lab maps module:")
        display_map(hablab_topdown_map)

        # easily save a map to file:
        map_filename = os.path.join(output_path, "top_down_map.png")
        imageio.imsave(map_filename, hablab_topdown_map)
# @markdown ## Querying the NavMesh

if not sim.pathfinder.is_loaded:
    print("Pathfinder not initialized, aborting.")
else:
    # @markdown NavMesh area and bounding box can be queried via *navigable_area* and *get_bounds* respectively.
    print("NavMesh area = " + str(sim.pathfinder.navigable_area))
    print("Bounds = " + str(sim.pathfinder.get_bounds()))

    # @markdown A random point on the NavMesh can be queried with *get_random_navigable_point*.
    pathfinder_seed = 1  # @param {type:"integer"}
    sim.pathfinder.seed(pathfinder_seed)
    nav_point = sim.pathfinder.get_random_navigable_point()
    print("Random navigable point : " + str(nav_point))
    print("Is point navigable? " + str(sim.pathfinder.is_navigable(nav_point)))

    # @markdown The radius of the minimum containing circle (with vertex centroid origin) for the isolated navigable island of a point can be queried with *island_radius*.
    # @markdown This is analogous to the size of the point's connected component and can be used to check that a queried navigable point is on an interesting surface (e.g. the floor), rather than a small surface (e.g. a table-top).
    print("Nav island radius : " + str(sim.pathfinder.island_radius(nav_point)))

    # @markdown The closest boundary point can also be queried (within some radius).
    max_search_radius = 2.0  # @param {type:"number"}
    print(
        "Distance to obstacle: "
        + str(sim.pathfinder.distance_to_closest_obstacle(nav_point, max_search_radius))
    )
    hit_record = sim.pathfinder.closest_obstacle_surface_point(
        nav_point, max_search_radius
    )
    print("Closest obstacle HitRecord:")
    print(" point: " + str(hit_record.hit_pos))
    print(" normal: " + str(hit_record.hit_normal))
    print(" distance: " + str(hit_record.hit_dist))

    vis_points = [nav_point]

    # HitRecord will have infinite distance if no valid point was found:
    if math.isinf(hit_record.hit_dist):
        print("No obstacle found within search radius.")
    else:
        # @markdown Points near the boundary or above the NavMesh can be snapped onto it.
        perturbed_point = hit_record.hit_pos - hit_record.hit_normal * 0.2
        print("Perturbed point : " + str(perturbed_point))
        print(
            "Is point navigable? " + str(sim.pathfinder.is_navigable(perturbed_point))
        )
        snapped_point = sim.pathfinder.snap_point(perturbed_point)
        print("Snapped point : " + str(snapped_point))
        print("Is point navigable? " + str(sim.pathfinder.is_navigable(snapped_point)))
        vis_points.append(snapped_point)

    # @markdown ---
    # @markdown ### Visualization
    # @markdown Running this cell generates a topdown visualization of the NavMesh with sampled points overlayed.
    meters_per_pixel = 0.1  # @param {type:"slider", min:0.01, max:1.0, step:0.01}

    if display:
        xy_vis_points = convert_points_to_topdown(
            sim.pathfinder, vis_points, meters_per_pixel
        )
        # use the y coordinate of the sampled nav_point for the map height slice
        top_down_map = maps.get_topdown_map(
            sim.pathfinder, height=nav_point[1], meters_per_pixel=meters_per_pixel
        )
        recolor_map = np.array(
            [[255, 255, 255], [128, 128, 128], [0, 0, 0]], dtype=np.uint8
        )
        top_down_map = recolor_map[top_down_map]
        print("\nDisplay the map with key_point overlay:")
        display_map(top_down_map, key_points=xy_vis_points)

# @markdown ## Pathfinding Queries on NavMesh

# @markdown The shortest path between valid points on the NavMesh can be queried as shown in this example.

# @markdown With a valid PathFinder instance:
if not sim.pathfinder.is_loaded:
    print("Pathfinder not initialized, aborting.")
else:
    seed = 4  # @param {type:"integer"}
    sim.pathfinder.seed(seed)

    # fmt off
    # @markdown 1. Sample valid points on the NavMesh for agent spawn location and pathfinding goal.
    # fmt on
    sample1 = sim.pathfinder.get_random_navigable_point()
    sample2 = sim.pathfinder.get_random_navigable_point()

    # @markdown 2. Use ShortestPath module to compute path between samples.
    path = habitat_sim.ShortestPath()
    path.requested_start = sample1
    path.requested_end = sample2
    found_path = sim.pathfinder.find_path(path)
    geodesic_distance = path.geodesic_distance
    path_points = path.points
    # @markdown - Success, geodesic path length, and 3D points can be queried.
    print("found_path : " + str(found_path))
    print("geodesic_distance : " + str(geodesic_distance))
    print("path_points : " + str(path_points))

    # @markdown 3. Display trajectory (if found) on a topdown map of ground floor
    if found_path:
        meters_per_pixel = 0.025
        scene_bb = sim.get_active_scene_graph().get_root_node().cumulative_bb
        height = scene_bb.y().min
        if display:
            top_down_map = maps.get_topdown_map(
                sim.pathfinder, height, meters_per_pixel=meters_per_pixel
            )
            recolor_map = np.array(
                [[255, 255, 255], [128, 128, 128], [0, 0, 0]], dtype=np.uint8
            )
            top_down_map = recolor_map[top_down_map]
            grid_dimensions = (top_down_map.shape[0], top_down_map.shape[1])
            # convert world trajectory points to maps module grid points
            trajectory = [
                maps.to_grid(
                    path_point[2],
                    path_point[0],
                    grid_dimensions,
                    pathfinder=sim.pathfinder,
                )
                for path_point in path_points
            ]
            grid_tangent = mn.Vector2(
                trajectory[1][1] - trajectory[0][1], trajectory[1][0] - trajectory[0][0]
            )
            path_initial_tangent = grid_tangent / grid_tangent.length()
            initial_angle = math.atan2(path_initial_tangent[0], path_initial_tangent[1])
            # draw the agent and trajectory on the map
            maps.draw_path(top_down_map, trajectory)
            maps.draw_agent(
                top_down_map, trajectory[0], initial_angle, agent_radius_px=8
            )
            print("\nDisplay the map with agent and path overlay:")
            display_map(top_down_map)

        # @markdown 4. (optional) Place agent and render images at trajectory points (if found).
        display_path_agent_renders = True  # @param{type:"boolean"}
        if display_path_agent_renders:
            print("Rendering observations at path points:")
            tangent = path_points[1] - path_points[0]
            agent_state = habitat_sim.AgentState()
            for ix, point in enumerate(path_points):
                if ix < len(path_points) - 1:
                    tangent = path_points[ix + 1] - point
                    agent_state.position = point
                    tangent_orientation_matrix = mn.Matrix4.look_at(
                        point, point + tangent, np.array([0, 1.0, 0])
                    )
                    tangent_orientation_q = mn.Quaternion.from_matrix(
                        tangent_orientation_matrix.rotation()
                    )
                    agent_state.rotation = utils.quat_from_magnum(tangent_orientation_q)
                    agent.set_state(agent_state)

                    observations = sim.get_sensor_observations()
                    rgb = observations["color_sensor"]
                    semantic = observations["semantic_sensor"]
                    depth = observations["depth_sensor"]

                    if display:
                        display_sample(rgb, semantic, depth)
```









