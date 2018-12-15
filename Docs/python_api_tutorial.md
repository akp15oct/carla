<h1>Python API tutorial</h1>

In this tutorial we introduce the basic concepts of the CARLA Python API, as
well as an overview of its most important functionalities. The reference of all
classes and methods available can be found at
[Python API reference](python_api.md).

!!! note
    **This document applies only to the latest development version**. <br>
    The API has been significantly changed in the latest versions starting at
    0.9.0. We commonly refer to the new API as **0.9.X API** as opposed to
    the previous **0.8.X API**.

First of all, we need to introduce a few core concepts:

  - **Actor:** Actor is anything that plays a role in the simulation and can be
    moved around, examples of actors are vehicles, pedestrians, and sensors.
  - **Blueprint:** Before spawning an actor you need to specify its attributes,
    and that's what blueprints are for. We provide a blueprint library with
    the definitions of all the actors available.
  - **World:** The world represents the currently loaded map and contains the
    functions for converting a blueprint into a living actor, among other. It
    also provides access to the road map and functions to change the weather
    conditions.

#### Connecting and retrieving the world

To connect to a simulator we need to create a "Client" object, and to do so we
need to provide the IP address and port of a running instance of the simulator

```py
client = carla.Client('localhost', 2000)
```

First thing most scripts do is setting the client time-out. This time-out sets a
time limit to all networking operations, if the time-out is not set networking
operations may block forever.

```py
client.set_timeout(10.0) # seconds
```

Once we have the client configured we can directly retrieve the world

```py
world = client.get_world()
```

Typically we won't need the client object anymore, all the objects created by
the world will connect to the IP and port provided if they need to. This
operations are usually done in the background and are transparent to the user.

#### Blueprints

A blueprint contains the information necessary to create a new actor. For
instance, if the blueprint defines a car, we can change its color here, if it
defines a lidar, we can decide here how many channels the lidar will have. A
blueprints also has an ID that uniquely identifies it and all the actor
instances created with it. Examples of IDs are "vehicle.nissan.patrol" or
"sensor.camera.depth".

The list of all available blueprints is kept in the **blueprint library**

```py
blueprint_library = world.get_blueprint_library()
```

The library allows us to find specific blueprints by ID, filter them with
wildcards, or just choosing one at random

```py
# Find specific blueprint.
collision_sensor_bp = blueprint_library.find('sensor.other.collision')
# Chose a vehicle blueprint at random.
vehicle_bp = random.choice(blueprint_library.filter('vehicle.bmw.*'))
```

Some of the attributes of the blueprints can be modified while some other are
just read-only. For instance, we cannot modify the number of wheels of a vehicle
but we can change its color

```py
vehicles = blueprint_library.filter('vehicle.*')
bikes = [x for x in vehicles if int(x.get_attribute('number_of_wheels')) == 2]
for bike in bikes:
    bike.set_attribute('color', '255,0,0')
```

Modifiable attributes also come with a list of recommended values

```py
for attr in blueprint:
    if attr.is_modifiable:
        blueprint.set_attribute(attr.id, random.choice(attr.recommended_values))
```

The blueprint system has been designed to ease contributors adding their custom
actors directly in Unreal Editor, we'll add a tutorial on this soon, keep tuned!

#### Spawning actors

Once we have the blueprint set up, spawning an actor is pretty straightforward

```py
transform = Transform(Location(x=230, y=195, z=40), Rotation(yaw=180))
actor = world.spawn_actor(blueprint, transform)
```

The spawn actor function comes in two flavours, `spawn_actor` and
`try_spawn_actor`. The former will raise an exception if the actor could not be
spawned, the later will return `None` instead. The most typical cause of
failure is collision at spawn point, meaning the actor does not fit at the spot
we chose; probably another vehicle is in that spot or we tried to spawn into a
building.

To ease the task of finding a spawn location, each map provides a list of
recommended transforms

```py
spawn_points = world.get_map().get_spawn_points()
```

We add more on the map class later in this tutorial.

Finally, the spawn functions have an optional argument that controls whether the
actor is going to be attached to another actor. This is specially useful for
sensors. In the next example the camera remains rigidly attached to our vehicle
during the rest of the simulation

```py
camera = world.spawn_actor(camera_bp, relative_transform, attach_to=my_vehicle)
```

Note that in this case, the transform provided is treated relative to the parent
actor.

#### Handling actors

Once we have an actor alive in the world, we can move this actor around and
check its dynamic properties

```py
location = actor.get_location()
location.z += 10.0
actor.set_location(location)
print(actor.get_acceleration())
print(actor.get_velocity())
```

We can even freeze and actor by disabling its physics simulation

```py
actor.set_simulate_physics(False)
```

And once we are tired of an actor we can remove it from the simulation with

```py
actor.destroy()
```

Note that actors are not cleaned up automatically when we the Python script
finishes, if we want to get rid of them we need to explicitly destroy them.

!!! important
    **Known issue:** To improve performance, most of the methods send requests
    to the simulator asynchronously. The simulator queues each of these
    requests, but only has a limited amount of time each update to parse them.
    If we flood the simulator by calling "set" methods too often, e.g.
    set_transform, the requests will accumulate a significant lag.

#### Vehicles

Vehicles are a special type of actors that provide a few methods specific for
wheeled vehicles. Apart from the handling methods common to all actors, vehicles
can also be controlled by providing throttle, break, and steer values

```py
vehicle.apply_control(carla.VehicleControl(throttle=1.0, steer=-1.0))
```

These are all the parameters of the VehicleControl object and their default
values

```py
carla.VehicleControl(
    throttle = 0.0
    steer = 0.0
    brake = 0.0
    hand_brake = False
    reverse = False
    manual_gear_shift = False
    gear = 0)
```

Our vehicles also come with a handy autopilot

```py
vehicle.set_autopilot(True)
```

As has been a common misconception, we need to clarify that this autopilot
control is purely hard-coded into the simulator and it's not based at all in
machine learning techniques.

Finally, vehicles also have a bounding box that encapsulates them

```py
box = vehicle.bounding_box
print(box.location)         # Location relative to the vehicle.
print(box.extent)           # XYZ half-box extents in meters.
```

#### Sensors

Sensors are a special type of actor that produce a stream of data. Sensors are
such a key component of CARLA that they deserve their own documentation page, so
here we'll limit ourselves to show a small example of how sensors work

```py
camera_bp = blueprint_library.find('sensor.camera.rgb')
camera = world.spawn_actor(camera_bp, relative_transform, attach_to=my_vehicle)
camera.listen(lambda image: image.save_to_disk('output/%06d.png' % image.frame_number))
```

In this example we have attached a camera to a vehicle, and told the camera to
save to disk each of the images generated.

The full list of sensors and their measurement is explained in
[Cameras and sensors](cameras_and_sensors.md).

#### Other actors

Apart from vehicles and sensors, there are many other actors in the world. The
full list can be requested to the world

```py
actor_list = world.get_actors()
```

The actor list object returned has functions for finding, filtering, and
iterating actors

```py
# Find an actor by id.
actor = actor_list.find(id)
# Print the location of all the speed limit signs in the world.
for speed_sign in actor_list.filter('traffic.speed_limit.*'):
    print(speed_sign.get_location())
```

Among the actors you can find in this list are

  * **Traffic lights** with a `state` property to check the light's current state.
  * **Speed limit signs** with the speed codified in their type_id.
  * The **Spectator** actor that can be used to move the view of the simulator window.

#### Changing the weather

The lighting and weather conditions can be requested and changed with the world
object

```py
weather = carla.WeatherParameters(
    cloudyness=80.0,
    precipitation=30.0,
    sun_altitude_angle=70.0)

world.set_weather(weather)

print(world.get_weather())
```

For convenience, we also provided a list of predefined weather presets that can
be directly applied to the world

```py
world.set_weather(carla.WeatherParameters.WetCloudySunset)
```

The full list of presets can be found in the
[WeatherParameters reference](python_api.md#carlaweatherparameters).

#### Map and waypoints

One of the key features of CARLA is that our roads are fully annotated, all our
maps come accompanied by an OpenDrive file that defines the road layout.
Furthermore, we provide a higher level API for querying and navigating this
information.

These objects were a recent addition to our API and are still in heavy
development, we hope to make them soon much more powerful that are now.

Let's start by getting the map of the current world

```py
map = world.get_map()
```

For starters, the map has a `name` attribute that matches the name of the
currently loaded city, e.g. Town01. And, as we've seen before, we can also ask
the map to provide a list of recommended locations for spawning vehicles,
`map.get_spawn_points()`.

However, the real power of this map API comes apparent when we introduce
waypoints. We can tell the map to give us a waypoint on the road closest to our
vehicle

```py
waypoint = map.get_waypoint(vehicle.get_location())
```

This waypoint's `transform` is located on a drivable lane, and it's oriented
according to the road direction at that point.

Waypoints also have function to query the "next" waypoints, this method returns
a list of waypoints at a certain distance that can be accessed from this
waypoint following the traffic rules. In other words, if a vehicle is placed in
this waypoint, give me the list of posible locations that this vehicle can drive
to. Let's see a practical example

```py
# Retrieve the closest waypoint.
waypoint = map.get_waypoint(vehicle.get_location())

# Disable physics, in this example we're just teleporting the vehicle.
vehicle.set_simulate_physics(False)

while True:
    # Find next waypoint 2 meters ahead.
    waypoint = random.choice(waypoint.next(2.0))
    # Teleport the vehicle.
    vehicle.set_transform(waypoint.transform)
```

The map object also provides methods for generating in bulk waypoints all over
the map at an approximated distance between them

```py
map.generate_waypoints(2.0)
```

For routing purposes, it is also possible to retrieve a topology graph of the
roads

```py
waypoint_tuple_list = map.get_topology()
```

this method returns a list of pairs (tuples) of waypoints, for each pair, the
first element connects with the second one. Only the minimal set of waypoints to
define the topology are generated by this method, only a waypoint for each lane
for each road segment in the map.

Finally, to allow access to the whole road information, the map object can be
converted to OpenDrive format, and saved to disk as such.