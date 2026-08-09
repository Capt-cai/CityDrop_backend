This is a living document. Changes are recorded in Change Logs

# Delivery Option path

## **DeliveryOptionController**

–– assembles address into string then calls DeliveryService.getDeliveryOptions  
–– frontend input: PackageRequestBody  
–– returns: array of DeliveryQuote

## **DeliveryService**

–– geocodes destinationAddress, pull stations using StationRepository, gets all order options  
–– methods:  
getDeliveryOptions(String destinationAddress, double packageWeightLbs)  
–– loops DeliveryAlgorithm for every combination of station and vehicle type  
–– use Google Maps API for geocoding  
–– returns: array of DeliveryQuote  
–– throw AddressCannotBeGeocodedException, 422

## **DeliveryAlgorithm**

–– calculates time and cost of delivery  
–– methods:  
computeTime(StationEntity station, double destCoordX, double destCoordY, String vehicle)  
–– returns: double time

computeCost(double packageWeightLbs, String vehicle)  
–– returns: double cost

## **StationRepository**

–– extends ListCrudRepository\<StationEntity, Integer\>  
–– methods:  
findByStationId(int stationId)  
–– returns: StationEntity

decrementRobotCount(int stationId)  
–– custom query needed, do not decrement if count==0  
–– returns: int rows affected

decrementDroneCount(int stationId)  
–– custom query needed, do not decrement if count==0  
–– returns: int rows affected

incrementRobotCount(int stationId)  
–– custom query needed  
–– returns: void

incrementDroneCount(int stationId)  
–– custom query needed  
–– returns: void

# Order path

## **OrderController**

–– find user by calling UserService.findByUsername(String username), document userId separately as it will be reused often  
–– frontend input: username (comes from the authenticated session, NOT literal request field)  
–– returns: UserEntity (not returned to frontend)

–– submit order by calling OrderService.submitOrder  
–– frontend input: SubmissionObject  
–– returns: OrderObject

–– retrieve order list by calling OrderService.listOrder  
–– frontend input: int userId  
–– returns: first array active, second array completed orders’ int orderId

–– retrieve particular order by calling OrderService.getOrder  
–– frontend input: int orderId  
–– returns: OrderObject

## **OrderService**

–– submit order, retrieve a user’s order list, retrieve a particular order  
–– methods:  
submitOrder(int userId, SubmissionObject order)  
–– validate vehicle availability by calling StationRepository.findByStationId  
–– if validated, calls DeliveryAlgorithm.computeCost to recompute price  
–– call OrderRepository.save to save the order  
–– decrement the according vehicle count of the according station in stations table  
 by calling StationRepository.decrementRobotCount/decrementDroneCount  
–– returns: OrderObject  
–– throw VehicleUnavailableException, 409, if rows affected==0

listOrder(int userId)  
–– for each active order, recompute its distance to destination, update its status and call StationRepository.incrementRobotCount/incrementDroneCount to increase the according vehicle’s count in the station it originated from  
–– call OrderRepository.findByUserIdAndStatusNot(... “delivered”) to retrieve active orders  
–– call OrderRepository.findByUserIdAndStatus(... “delivered”) to retrieve delivered orders  
–– retrieve orderId from each Order object  
–– returns: two arrays of int orderId of active and completed orders respectively

getOrder(int userId, int orderId)  
–– for the order of orderId, recompute its distance to destination and update its status and call StationRepository.incrementRobotCount/incrementDroneCount to increase the according vehicle’s count in the station it originated from  
–– calls OrderRepository.findByUserIdAndOrderId  
–– returns: OrderObject  
–– throws OrderNotFoundException, 404

## **OrderRepository**

–– extends ListCrudRepository\<OrderEntity, Integer\>  
–– methods:  
findByUserIdAndStatusNot(int userId, String status)  
–– returns: List\<OrderEntity\>

findByUserIdAndStatus(int userId, String status)  
–– returns: List\<OrderEntity\>

findByUserIdAndOrderId(int userId, int orderId)  
–– returns: OrderEntity

# User path

## **UserController**

–– calls UserService.register  
–– frontend input: RegisterBody  
–– returns: void

## **UserService**

–– registers user, finds user (reference UserService in Twitch)  
–– methods:  
register(String username, String password)  
–– userDetailsManager.userExists(username) to check if the username is taken  
–– if not, create UserDetails user with passwordEncoder.encode(password), role as user, then userDetailsManager.createUser(user)  
–– returns: void  
–– throw UsernameTakenException, 409

findByUsername(String username)  
–– returns: UserEntity

## **UserRepository**

–– extends ListCrudRepository\<UserEntity, Integer\>  
–– methods:  
findByUsername(String username)  
–– returns: UserEntity

# Security config

permitAll() paths: /delivery-options, /register, /login, /logout

formlogin()/logout(): Spring Security’s built in methods, no custom controllers needed  
–– custom success and failure handlers on formlogin() that overrides Spring’s default error messages. These must be separate from GlobalExceptionHandler as they will be hit before the entire app  
–– HttpStatusReturningLogoutSuccessHandler(NO\_CONTENT) on logout()

UserDetailsManager bean returns a JdbcUserDetailsManager, PasswordEncoder bean returns PasswordEncoderFactories.createDelegatingPasswordEncoder()

(reference Twitch’s AppConfig)

# Tables

**orders**: int orderId, int userId, String destination, double packageWeightLbs, double price, String vehicle, int stationId, String status, String createdAt  
**stations**: int stationId, double coordX, double coordY, double radius, int robotCount, int droneCount  
**users**: int id, String username, String password, boolean enabled (default 1\)  
**authorities**: int id, String username, String authority

# Exceptions

GlobalExceptionHandler file: matches each exception’s status code to an error message

individual exception files: AddressCannotBeGeocodedException, VehicleUnavailableException, OrderNotFoundException, UsernameTakenException

# Models

## Entities

**OrderEntity**: int orderId, int userId, String destination, double packageWeightLbs, double price, String vehicle, int stationId, String status, String createdAt  
**StationEntity**: int stationId, double coordX, double coordY, double radius, int robotCount, int droneCount  
**User**: int id, String username, String password, boolean enabled  
Authorities: int id, String username, String authority

## Request Models

**RegisterBody**: String username, String password  
**PackageRequestBody**: String destStreet, String destCity, String destState, String destZip, double packageWeight  
**SubmissionObject**: String destination, double packageWeightLbs, int stationId, String vehicle

## Response Models

**DeliveryQuote**: String destination, double packageWeightLbs, String vehicle, double price, double time, int stationId  
**OrderObject**: int orderId, String destination, double packageWeightLbs, double price, double time, String vehicle, int stationId, String status, String createdAt  
**OrderListResponse**: active: List\<OrderIdEntry\>, completed: List\<OrderIdEntry\>  
**OrderIdEntry**: int orderId

# Enums

**VehicleType**: ROBOT, DRONE  
**OrderStatus**: PENDING\_DROPOFF, AT\_STATION, BEFORE\_HALF\_WAY, HALF\_WAY, MORE\_THAN\_HALF\_WAY, DELIVERED

# Change Logs

# Known Bugs

1. vehicle updating mechanism may cause bugs where if the user never checks the id, it will simply never update. This may lead to all orders in fact completed when no vehicles are available. To fix this, the server probably needs some kind of timer for each order, which I simply haven’t figured out how to do with a local dev environment that gets constantly restarted.