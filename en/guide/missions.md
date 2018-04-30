# Missions

The DroneCore Mission API allows you to create, upload, run, pause, restart, jump to item in, and track missions. Missions can have multiple "mission items", each which may specify a position, altitude, fly-through behaviour, camera action, gimbal position, and the speed to use when traveling to the next position.

> **Note** The API enables a small but useful subset of the mission commands supported by PX4 (and the MAVLink specification). For example, it does not currently support "repeat", takeoff, return to land etc.

Missions are *managed* though the [Mission](../api_reference/classdronecore_1_1_mission.md) class, which communicates with the device to upload mission information and run, pause, track the mission progress etc. The mission that is uploaded to the vehicle is defined as a vector of [MissionItem](../api_reference/classdronecore_1_1_mission_item.md) objects. 


## Preconditions

The following code assumes that you already have included DroneCore (`#include <dronecore/dronecore.h>`) and the standard library (`#include <functional>`) and that there is a [connection to a device](../guide/connections.md) obtained as shown below:
```cpp
Device &device = dc.device(); 
```

## Defining a Mission

A mission must be defined as a vector of [MissionItem](../api_reference/classdronecore_1_1_mission_item.md) objects as shown below:
```cpp
std::vector<std::shared_ptr<MissionItem>> mission_items;
```

You can create as many `MissionItem` objects as you like and use `std_vector::push_back()` to add them to the back of the mission item vector. The example below shows how to create and add a `MissionItem`, that just sets the target position (using [set_position()](../api_reference/classdronecore_1_1_mission_item.md#classdronecore_1_1_mission_item_1ab5897670c8830fc3514036d6ee99b582)). 
```cpp
// Create MissionItem and set its position
std::shared_ptr<MissionItem> new_item(new MissionItem());
new_item->set_position(47.40 /*latitude_deg*/, 8.5455360114574432 /*longitude_deg*/);

// Add new_item to the vector
mission_items.push_back(new_item);
```

The example below shows how you might set the other options on a second `MissionItem` (and add it to the mission).

```cpp
std::shared_ptr<MissionItem> new_item2(new MissionItem());
new_item2->set_position(47.50 /*latitude_deg*/, 8.5455360114574432 /*longitude_deg*/);
new_item2->set_relative_altitude(2.0f);
new_item2->set_speed(5.0f /* m/s */);
new_item2->set_fly_through(true);
new_item2->set_gimbal_pitch_and_yaw(20.0f /*pitch*/, 60.0f /*yaw degrees*/);
new_item2->set_camera_action(MissionItem::CameraAction::TAKE_PHOTO);
    
//Add new_item2 to the vector
mission_items.push_back(new_item2);
```

> **Note** The autopilot has sensible default values for the attributes. If you do set a value (e.g. the desired speed) 
then it will be the default for the remainder of the mission.

<span></span>
> **Note** There are also getter methods for querying the current value of `MissionItem` attributes. 
The default values of most fields are `NaN` (which means they are ignored/not sent).


### Convenience Function

The [Fly Mission](../examples/fly_mission.md) uses a convenience function to create `MissionItem` objects. 
Using this approach you have to specify every attribute for every mission item, whether or not the 
value is actually used.

The definition and use of this function is shown below:

```cpp
std::shared_ptr<MissionItem> add_mission_item(double latitude_deg,
                                              double longitude_deg,
                                              float relative_altitude_m,
                                              float speed_m_s,
                                              bool is_fly_through,
                                              float gimbal_pitch_deg,
                                              float gimbal_yaw_deg,
                                              MissionItem::CameraAction camera_action)
{
    std::shared_ptr<MissionItem> new_item(new MissionItem());
    new_item->set_position(latitude_deg, longitude_deg);
    new_item->set_relative_altitude(relative_altitude_m);
    new_item->set_speed(speed_m_s);
    new_item->set_fly_through(is_fly_through);
    new_item->set_gimbal_pitch_and_yaw(gimbal_pitch_deg, gimbal_yaw_deg);
    new_item->set_camera_action(camera_action);
    return new_item;
}
    
    
mission_items.push_back(
    add_mission_item(47.398170327054473,
                     8.5456490218639658,
                     10.0f, 5.0f, false,
                     20.0f, 60.0f,
                     MissionItem::CameraAction::NONE));
```


## Uploading a Mission

Use [Mission::send_mission_async()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1ad06a0751e303eb5c1f1ba37d3b9ede47) to upload the mission defined in the previous section.

The example below shows how this is done, using promises to wait on the result.

```cpp
// ... declare and populate the mission vector: mission_items

{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();
    device.mission().send_mission_async(
    mission_items, [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get();
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Mission upload failed (" << Mission::result_str(result) << "), exiting." << std::endl;
        return 1;
    }
    std::cout << "Mission uploaded." << std::endl;
}
```

## Starting/Pausing Missions 

Start or resume a paused mission using [Mission::start_mission_async()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1a9e032c6b2bc35cf6e7e19e07747fb0d3). The vehicle must already have a mission (the mission need not have been uploaded through *DroneCore*).

The code fragment below shows how this is done, using promises to wait on the result.

```cpp
{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();
    device.mission().start_mission_async(
    [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get(); //Wait on result
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Mission start failed (" << Mission::result_str(result) << "), exiting." << std::endl;
        return 1;
    }
    std::cout << "Started mission." << std::endl;
}
```

To pause a mission use [Mission::pause_mission_async()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1a65f729cf954586507ecd8dc07a510dd1). The code is almost exactly the same as for starting a mission:

```cpp
{
    auto prom = std::make_shared<std::promise<Mission::Result>>();
    auto future_result = prom->get_future();

    std::cout << "Pausing mission..." << std::endl;
    device.mission().pause_mission_async(
    [prom](Mission::Result result) {
        prom->set_value(result);
    });

    const Mission::Result result = future_result.get();
    if (result != Mission::Result::SUCCESS) {
        std::cout << "Failed to pause mission (" << Mission::result_str(result) << ")" << std::endl;
    } else {
        std::cout << "Mission paused." << std::endl;
    }
}
```

## Monitoring Progress

Asynchronously monitor progress using [Mission::subscribe_progress()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1a3290fc79eb22f899528328adfca48a61), which receives a regular callback with the current `MissionItem` number and the total number of items.

The code fragment just takes a lambda function that reports the current status. 

```cpp
device.mission().subscribe_progress( [](int current, int total) {
       std::cout << "Mission status update: " << current << " / " << total << std::endl;
    });
```

> **Note** The mission is complete when `current == total`.

The following synchronous methods are also available for checking mission progress:
* [mission_finished()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1abf3463efaa18147a1c179e7449503829) - Checks if mission has been finished.
* [current_mission_item()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1a1faa448b32cd0028923b22de0cc78e9c) - Returns the current mission item index.
* [total_mission_items()](../api_reference/classdronecore_1_1_mission.md#classdronecore_1_1_mission_1a9d2195ec1af301c51002f8cb99aa22e9) - Gets the total number of items.

> **Note** The mission is (also) complete when `current_mission_item()` == `total_mission_items()`.


## Taking off, Landing, Returning

If using a copter or VTOL vehicle then PX4 will automatically takeoff when it is armed and a mission is started (even without a takeoff mission item). For Fixed Wing vehicles the vehicle must be launched before starting a mission.

At time of writing the Mission API does not provide takeoff, land or "return to launch" `MissionItems`. If required you can instead use the appropriate commands in the [Action](../api_reference/classdronecore_1_1_action.md) class.

<!-- when we have a topic on taking off and landing in the guide, link to that instead -->
<!-- Update if we get new mission items -->


## Further Information

* [Mission Flight Mode](https://docs.px4.io/en/flight_modes/mission.html) (PX4 User Guide)
* [Fly Mission](../examples/fly_mission.md) (DroneCore Example)
* Integration tests:
  * [mission.cpp](https://github.com/dronecore/DroneCore/blob/v0.3.0/integration_tests/mission.cpp)
  * [mission_change_speed.cpp](https://github.com/dronecore/DroneCore/blob/v0.3.0/integration_tests/mission_change_speed.cpp)
  * [mission_survey.cpp](https://github.com/dronecore/DroneCore/blob/v0.3.0/integration_tests/mission_survey.cpp)
