Custom Service Tutorial
-------

This tutorial will show you how to create a custom service with a custom value characteristic in the ble_app_template project found in the Nordic nRF5 SDK v13.0.0. This tutorial can be seen as the combined version of the BLE [Advertising](https://devzone.nordicsemi.com/tutorials/5) / [Services](https://devzone.nordicsemi.com/tutorials/8) / [Characteristics](https://devzone.nordicsemi.com/tutorials/17) , A Beginner's Tutorial series, which I strongly recommend to take a look at as they go deeper into the matter than this tutorial.

The aim of this tutorial is simply to create one service with one characteristic without too much theory in between the steps. There are no .c or .h files that needs to be downloaded as we will be starting from scratch in the ble_app_template project. 

<!---
## TODO

- [ ] Add register definition file (.svd) and retarget of printf to the ble_app_uart Segger Embedded Project.
--->

## Requirements

- nRF5 SDK v13.0.0
- nRF52 DK
- Keil ARM MKD v5.22
- nRF Commandline Tools

## Tutorial Steps
<!---
```c    
    uint8_t data
    // put C code here
```

[link to webpage](www.google.com)
--->
### Step 1 

The first thing we need to do is to create a new .c file, lets call it ble_custom_service.c, and its accompaning .h file ble_custom_service.h. We'll start by declaring types and functions in ble_custom_service.h before we move to the actual implementation of our custom service. At the top we'll need to include the following .h files

```c
#include <stdint.h>
#include <stdbool.h>
#include "ble.h"
#include "ble_srv_common.h"
```

Next, we're going to need a 128-bit UUID for our custom service since its not Bluetooth SIG . There are several ways to generate a 128-bit UUID, but we'll use [this](https://www.uuidgenerator.net/version4) Online UUID generator. The webpage will generate a random 128-bit UUID, which in my case was

```
f364adc9-b000-4042-ba50-05ca45bf8abc
```
The UUID is given as the sixteen octets of a UUID are represented as 32 hexadecimal (base 16) digits, displayed in five groups separated by hyphens, in the form 8-4-4-4-12. The 16 octets are given in big-endian, while we use the small-endian representation in our SDK. Thus, we must reverse the byte-ordering when we define our UUID base, as shown below.

```c
#define CUSTOM_SERVICE_UUID_BASE         {0xBC, 0x8A, 0xBF, 0x45, 0xCA, 0x05, 0x50, 0xBA, \
                                          0x40, 0x42, 0xB0, 0x00, 0xC9, 0xAD, 0x64, 0xF3}
```
Now that we have defined our Base UUID, we need to define a 16-bit UUID for the Custom Service and a 16-bit UUID for a Custom Value Characteristic.  

```c
#define CUSTOM_SERVICE_UUID               0x1400
#define CUSTOM_VALUE_CHAR_UUID            0x1401
```
The values for the 16-bit UUIDs can pretty much be choosen by random 

- [ ] Look up if the 16-bit UUIDs can be chosen randomly or if any rules apply. 

Next we need to declare an event type specific to our service

```c
typedef enum
{
    BLE_CUS_EVT_NOTIFICATION_ENABLED,                             /**< Custom value notification enabled event. */
    BLE_CUS_EVT_NOTIFICATION_DISABLED,                             /**< Custom value notification disabled event. */
    BLE_CUS_EVT_DISCONNECTED,
    BLE_CUS_EVT_CONNECTED
} ble_cus_evt_type_t;
```

After declaring the event type we need to declare an event structure that holds a ble_cus_evt_type_t event, i.e. 

```c
/**@brief Custom Service event. */
typedef struct
{
    ble_cus_evt_type_t evt_type;                                  /**< Type of event. */
} ble_cus_evt_t;
```

The next step is to add a forward declaration of the ble_cus_t type
```c
// Forward declaration of the ble_cus_t type.
typedef struct ble_cus_s ble_cus_t;
```
The reason why we need to do this is because the ble_cus_t type will be referenced in the decleration of the Custom Service event handler type which we will define next

```c
/**@brief Custom Service event handler type. */
typedef void (*ble_cus_evt_handler_t) (ble_cus_t * p_bas, ble_cus_evt_t * p_evt);
```

This event handler type can be used to declare an event handler function that we can invoke and pass ble_cus_evt_type_t events to. More on this latter. 

Ok, so far so good. Now we need to create two structures, one Custom Service init structure, ble_cus_init_t struct to hold all the options and data needed to initialize our custom service.

```c
/**@brief Battery Service init structure. This contains all options and data needed for
 *        initialization of the service.*/
typedef struct
{
    ble_cus_evt_handler_t         evt_handler;                    /**< Event handler to be called for handling events in the Custom Service. */
    uint8_t                       initial_custom_value;           /**< Initial custom value */
    ble_srv_cccd_security_mode_t  custom_value_char_attr_md;     /**< Initial security level for Custom characteristics attribute */
} ble_cus_init_t;
```

The second struct that we need to create is the Custom Service structure, ble_cus_s, which holds the status information of the service. 

```c
/**@brief Custom Service structure. This contains various status information for the service. */
struct ble_cus_s
{
    ble_cus_evt_handler_t         evt_handler;                    /**< Event handler to be called for handling events in the Custom Service. */
    uint16_t                      service_handle;                 /**< Handle of Custom Service (as provided by the BLE stack). */
    ble_gatts_char_handles_t      custom_value_handles;           /**< Handles related to the Custom Value characteristic. */
    uint16_t                      conn_handle;                    /**< Handle of the current connection (as provided by the BLE stack, is BLE_CONN_HANDLE_INVALID if not in a connection). */
    uint8_t                       uuid_type; 
};
```


Great, the last thing we need to do in the ble_custom_service.h file is to add some function declerations. First, we're going to add the ble_cus_init function, which we're going to initialize our service with.

```c
/**@brief Function for initializing the Custom Service.
 *
 * @param[out]  p_cus       Custom Service structure. This structure will have to be supplied by
 *                          the application. It will be initialized by this function, and will later
 *                          be used to identify this particular service instance.
 * @param[in]   p_cus_init  Information needed to initialize the service.
 *
 * @return      NRF_SUCCESS on successful initialization of service, otherwise an error code.
 */
uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init);
```
Second, we're going to add the ble_cus_on_ble_evt function to handle the events of the ble_cus_evt_type_t from our service.

```c
/**@brief Function for handling the Application's BLE Stack events.
 *
 * @details Handles all events from the BLE stack of interest to the Battery Service.
 *
 * @note 
 *
 * @param[in]   p_cus      Custom Service structure.
 * @param[in]   p_ble_evt  Event received from the BLE stack.
 */
void ble_cus_on_ble_evt(ble_cus_t * p_bas, ble_evt_t * p_ble_evt);
```

Lastly we're going to add the ble_cus_custom_value_update function, which we're going to use to update our Custom Value Characteristic.

```c
/**@brief Function for updating the custom value.
 *
 * @details The application calls this function when the cutom value should be updated. If
 *          notification has been enabled, the custom value characteristic is sent to the client.
 *
 * @note 
 *       
 * @param[in]   p_bas          Custom Service structure.
 * @param[in]   Custom value 
 *
 * @return      NRF_SUCCESS on success, otherwise an error code.
 */

uint32_t ble_cus_custom_value_update(ble_cus_t * p_cus, uint8_t custom_value);
```

### Step 2 - Implementing the Custom Service 

First things first, we need to include the ble_custom_service.h header file we just created as well as some common SDK header files in ble_custom_service.c.

```c
#include "sdk_common.h"
#include "ble_srv_common.h"
#include "ble_cus.h"
#include <string.h>
```

The first function we're going to implement is ble_cus_init function. The first thing we should do upon entering ble_cus_init is to check that none of the pointers that we passed as arguments are NULL and declare the two variables err_code and ble_uuid.

```c
uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    uint32_t   err_code;
    ble_uuid_t ble_uuid;
}
```
After verifying that the pointers are valid we can initialize the Custom Service structure

```c
// Initialize service structure
p_cus->evt_handler               = p_cus_init->evt_handler;
p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;
```

This consists of setting the connection handle to invalid( should only be valid when we're in a connection) and providing a function pointer to the custom service event handler. Next, we're going to add our custom (also referred to as vendor specific) base UUID to the BLE stack's table.

```c
// Add Custom Service UUID
ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
err_code =  sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
VERIFY_SUCCESS(err_code);

ble_uuid.type = p_cus->uuid_type;
ble_uuid.uuid = CUSTOM_SERVICE_UUID;
```

We're almost done, the last thing we have to do is to add the Custom Service decleration to the BLE Stack's GATT table. 

```c
// Add the Custom Service
err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
if (err_code != NRF_SUCCESS)
{
    return err_code;
}
```

If you have followed the steps correctly, then ble_cus_init should look like this.

```c
uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    uint32_t   err_code;
    ble_uuid_t ble_uuid;

    // Initialize service structure
    p_cus->evt_handler               = p_cus_init->evt_handler;
    p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;

    // Add Custom Service UUID
    ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
    err_code =  sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
    VERIFY_SUCCESS(err_code);
    
    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_SERVICE_UUID;

    // Add the Custom Service
    err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }
}
```

### Step 3 - Initializing the Service and advertising our 128-bit UUID.

Now it's time to initialize the service in main.c and put our 128-bit UUID in the advertisement packet so that other BLE devices can see that our device has the Custom Service. 

First, add the ble_cus.h file to the include list in main.c

```c
#include "ble_cus.h"
```

and then add the m_cus of the ble_cus_t type below the defines 

```c
static ble_cus_t    m_cus;
``` 

The next step is to find the empty services_init function in main.c, which should look like this

```c
/**@brief Function for initializing services that will be used by the application.
 */
static void services_init(void)
{
    /* YOUR_JOB: Add code to initialize the services used by the application.
       ret_code_t                         err_code;
       ble_xxs_init_t                     xxs_init;
       ble_yys_init_t                     yys_init;

       // Initialize XXX Service.
       memset(&xxs_init, 0, sizeof(xxs_init));

       xxs_init.evt_handler                = NULL;
       xxs_init.is_xxx_notify_supported    = true;
       xxs_init.ble_xx_initial_value.level = 100;

       err_code = ble_bas_init(&m_xxs, &xxs_init);
       APP_ERROR_CHECK(err_code);

       // Initialize YYY Service.
       memset(&yys_init, 0, sizeof(yys_init));
       yys_init.evt_handler                  = on_yys_evt;
       yys_init.ble_yy_initial_value.counter = 0;

       err_code = ble_yy_service_init(&yys_init, &yy_init);
       APP_ERROR_CHECK(err_code);
     */
}
```

Ok, we're going to do as we're told, i.e. create a ble_cus_init_t struct and populate it with the necessary data and then pass it as an argument to our service init function ble_cus_init();

```c
/**@brief Function for initializing services that will be used by the application.
 */
static void services_init(void)
{
    /* YOUR_JOB: Add code to initialize the services used by the application.*/
    ret_code_t                         err_code;
    ble_cus_init_t                     cus_init;

     // Initialize CUS Service init structure to zero.
    memset(&cus_init, 0, sizeof(cus_init));
    
    // Set the cus event handler
    cus_init.evt_handler                = NULL;
	
    err_code = ble_cus_init(&m_cus, &cus_init);
    APP_ERROR_CHECK(err_code);	
}
```
We're setting the event handler to NULL as we have not created a event handler function yet, but do not worry we will do that in one of the steps ahead.Now that we have initialized the service we have to add the 128bit UUID to the advertisement packet. If you navigate to the top of main.c you should find the m_adv_uuids array.

```c
// YOUR_JOB: Use UUIDs for service(s) used in your application.
static ble_uuid_t m_adv_uuids[] = {{BLE_UUID_DEVICE_INFORMATION_SERVICE, BLE_UUID_TYPE_BLE}}; /**< Universally unique service identifiers. */
```

We need to replace the BLE_UUID_DEVICE_INFORMATION_SERVICE with the CUSTOM_SERVICE_UUID we defined in ble_custom_service.h as well as replace  BLE_UUID_TYPE_BLE with BLE_UUID_TYPE_VENDOR_BEGIN since this is a 128-bit vendor specific UUID and not a 16-bit Bluetooth SIG UUDID. m_adv_uuids should now look like this

```c
// YOUR_JOB: Use UUIDs for service(s) used in your application.
 ble_uuid_t m_adv_uuids[] = {{CUSTOM_SERVICE_UUID, BLE_UUID_TYPE_VENDOR_BEGIN }}; /**< Universally unique service identifiers. */
```

The final step we have to do is to change the calling order in  main() so that services_init() is called before advertising_init(). This is because we need to add the CUSTOM_SERVICE_UUID_BASE to the BLE stack's table using sd_ble_uuid_vs_add() in ble_cus_init() before we call advertising_init(). Doing it the otherway around will cause advertising_init() to return an error code. 

That should be it, compile the ble_app_template project, flash the S132 v4.0.2 SoftDevice and then flash the ble_app_template application. LED1 on your nRF52 DK should now start blinking, indicating that its advertising.  Use nRF Connect for Android/iOS to view the content of the advertisment package. 


- [ ] Verify that attempting to connect is possible all though we have not implemented any event handling. 

### Step 4 - Adding a Custom Value Characteristic to the Custom Service.

A service is nothing with out a characteristic, so lets add one of those by creating the custom_value_char_add function that we declared in the header file. The first thing we have to do is to add several metadata variables that we will later populate. 

```c
/**@brief Function for adding the Custom Value characteristic.
 *
 * @param[in]   p_cus        Custom Service structure.
 * @param[in]   p_cus_init   Information needed to initialize the service.
 *
 * @return      NRF_SUCCESS on success, otherwise an error code.
 */
static uint32_t custom_value_char_add(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    uint32_t            err_code;
    ble_gatts_char_md_t char_md;
    ble_gatts_attr_md_t cccd_md;
    ble_gatts_attr_t    attr_char_value;
    ble_uuid_t          ble_uuid;
    ble_gatts_attr_md_t attr_md;
}
```
Now starts the rather tedious part of populating all these variables, we'll start with char_md, which sets the properties that will be displayed to the central during service discovery.

```c
    memset(&char_md, 0, sizeof(char_md));

    char_md.char_props.read   = 1;
    char_md.char_props.write  = 1;
    char_md.char_props.notify = 0; 
    char_md.p_char_user_desc  = NULL;
    char_md.p_char_pf         = NULL;
    char_md.p_user_desc_md    = NULL;
    char_md.p_cccd_md         = NULL; 
    char_md.p_sccd_md         = NULL;
}
```
So we want to be able to both write and read to our Custom Value characteristic, but we do not want to enable the notify property until later. Next we're going to populate the attr_md, which actually sets the properties( i.e. accessability of the attribute).

```c
    memset(&attr_md, 0, sizeof(attr_md));

    attr_md.read_perm  = p_cus_init->custom_value_char_attr_md.read_perm;
    attr_md.write_perm = p_cus_init->custom_value_char_attr_md.write_perm;
    attr_md.vloc       = BLE_GATTS_VLOC_STACK;
    attr_md.rd_auth    = 0;
    attr_md.wr_auth    = 0;
    attr_md.vlen       = 0;
}
```  

The permissions set in the attr_md struct should correspond with the properties set in the characteristic metadata struct char_md. We're going to provide the permissions in the Custom Service init structure that we pass to ble_cus_init in services_init(). The .vloc option is set to BLE_GATTS_VLOC_STACK as we want the characteristic to be stored in the SoftDevice RAM section and not in the Application RAM section. 

The next variable that we have to populate is ble_uuid, which is going to hold our CUSTOM_VALUE_CHAR_UUID and is of the same type as the CUSTOM_SERVICE_UUID_BASE, i.e. vendor specific, which we specified in the .uuid_type field of Custom Service strucure when we added the CUSTOM_SERVICE_UUID_BASE to the BLE stack's table.

```c
    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_VALUE_CHAR_UUID;
```

Next, we're going to populate the attr_char_value struct, which sets the UUID, points to the attribute metadata and sets the size of the characteristic, in our case a single byte(uint8_t). 

```c
    memset(&attr_char_value, 0, sizeof(attr_char_value));

    attr_char_value.p_uuid    = &ble_uuid;
    attr_char_value.p_attr_md = &attr_md;
    attr_char_value.init_len  = sizeof(uint8_t);
    attr_char_value.init_offs = 0;
    attr_char_value.max_len   = sizeof(uint8_t);
```

Finally, we're done populating structs and we can add our characteristic by calling sd_ble_gatts_characteristic_add() with the structs as arguments.

```c
err_code = sd_ble_gatts_characteristic_add(p_cus->service_handle, &char_md,
                                               &attr_char_value,
                                               &p_cus->custom_value_handles);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    return NRF_SUCCESS;
```

After all that hard work (copy-pasting) your custom_value_char_add() function should look like this. 

```c
static uint32_t custom_value_char_add(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    uint32_t            err_code;
    ble_gatts_char_md_t char_md;
    ble_gatts_attr_md_t cccd_md;
    ble_gatts_attr_t    attr_char_value;
    ble_uuid_t          ble_uuid;
    ble_gatts_attr_md_t attr_md;

    memset(&char_md, 0, sizeof(char_md));

    char_md.char_props.read   = 1;
    char_md.char_props.write  = 1;
    char_md.char_props.notify = 0; 
    char_md.p_char_user_desc  = NULL;
    char_md.p_char_pf         = NULL;
    char_md.p_user_desc_md    = NULL;
    char_md.p_cccd_md         = NULL; 
    char_md.p_sccd_md         = NULL;
		
    memset(&attr_md, 0, sizeof(attr_md));

    attr_md.read_perm  = p_cus_init->custom_value_char_attr_md.read_perm;
    attr_md.write_perm = p_cus_init->custom_value_char_attr_md.write_perm;
    attr_md.vloc       = BLE_GATTS_VLOC_STACK;
    attr_md.rd_auth    = 0;
    attr_md.wr_auth    = 0;
    attr_md.vlen       = 0;

    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_VALUE_CHAR_UUID;

    memset(&attr_char_value, 0, sizeof(attr_char_value));

    attr_char_value.p_uuid    = &ble_uuid;
    attr_char_value.p_attr_md = &attr_md;
    attr_char_value.init_len  = sizeof(uint8_t);
    attr_char_value.init_offs = 0;
    attr_char_value.max_len   = sizeof(uint8_t);

    err_code = sd_ble_gatts_characteristic_add(p_cus->service_handle, &char_md,
                                               &attr_char_value,
                                               &p_cus->custom_value_handles);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    return NRF_SUCCESS;
}
```
 The final step is to call custom_value_char_add() at the end of ble_cus_init the service has been added, i.e. 

```c
uint32_t ble_cus_init(ble_cus_t * p_cus, const ble_cus_init_t * p_cus_init)
{
    if (p_cus == NULL || p_cus_init == NULL)
    {
        return NRF_ERROR_NULL;
    }

    uint32_t   err_code;
    ble_uuid_t ble_uuid;

    // Initialize service structure
    p_cus->evt_handler               = p_cus_init->evt_handler;
    p_cus->conn_handle               = BLE_CONN_HANDLE_INVALID;

    // Add Custom Service UUID
    ble_uuid128_t base_uuid = {CUSTOM_SERVICE_UUID_BASE};
    err_code =  sd_ble_uuid_vs_add(&base_uuid, &p_cus->uuid_type);
    VERIFY_SUCCESS(err_code);
    
    ble_uuid.type = p_cus->uuid_type;
    ble_uuid.uuid = CUSTOM_SERVICE_UUID;

    // Add the Custom Service
    err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &ble_uuid, &p_cus->service_handle);
    if (err_code != NRF_SUCCESS)
    {
        return err_code;
    }

    // Add Custom Value characteristic
    return custom_value_char_add(p_cus, p_cus_init);
}
```


### Step X - Handling events from the SoftDevice.
Great, we now have a Custom Service and a Custom Value Characteristic, but we want to be able to write to the characteristic and perform a specific task based on the value that was written to the characteristic, e.g. turn on a LED. However, before we can do that we need to do some event handling in ble_custom_service.c. We're now going to implement the ble_cus_on_ble_evt event handler that we declared in the header file.  

Upon entry its considered good practice to check that none of the pointers that we provided as arguments are NULL. 

```c
void ble_cus_on_ble_evt(ble_cus_t * p_cus, ble_evt_t * p_ble_evt)
{
    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
}
```

After the NULL check we're going to add a switch-case to check which event that has been propagated to the application by the SoftDevice. 

```c
void ble_cus_on_ble_evt(ble_cus_t * p_cus, ble_evt_t * p_ble_evt)
{
    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
    
    switch (p_ble_evt->header.evt_id)
    {
        case BLE_GAP_EVT_CONNECTED:
            break;

        case BLE_GAP_EVT_DISCONNECTED:
            break;

        default:
            // No implementation needed.
            break;
    }
}
```

For now we only need to care about the BLE_GAP_EVT_CONNECTED and BLE_GAP_EVT_DISCONNECTED events. So let's create the two functions on_connect() and on_disconnect(), starting with on_connect(). The only thing we need to do when we get the Connect event is to assign the connection handle in the Custom Service structure to the connection handle that is passed with the event. 

```c
/**@brief Function for handling the Connect event.
 *
 * @param[in]   p_cus       Custom Service structure.
 * @param[in]   p_ble_evt   Event received from the BLE stack.
 */
static void on_connect(ble_cus_t * p_cus, ble_evt_t * p_ble_evt)
{
    p_cus->conn_handle = p_ble_evt->evt.gap_evt.conn_handle;
}
```

Similarly, when we get the Disconnect event, the only thing we need to do is invalidate the connection handle in the Custom Service structure since the connection is now dead.

```c
/**@brief Function for handling the Disconnect event.
 *
 * @param[in]   p_cus       Custom Service structure.
 * @param[in]   p_ble_evt   Event received from the BLE stack.
 */
static void on_disconnect(ble_cus_t * p_cus, ble_evt_t * p_ble_evt)
{
    UNUSED_PARAMETER(p_ble_evt);
    p_cus->conn_handle = BLE_CONN_HANDLE_INVALID;
}
```

Now that we have one function for each event we only need to call the function in ble_cus_on_ble_evt, i.e.

```c
void ble_cus_on_ble_evt(ble_cus_t * p_cus, ble_evt_t * p_ble_evt)
{
    if (p_cus == NULL || p_ble_evt == NULL)
    {
        return;
    }
    
    switch (p_ble_evt->header.evt_id)
    {
        case BLE_GAP_EVT_CONNECTED:
            on_connect(ble_cus_t * p_cus, ble_evt_t * p_ble_evt);
            break;

        case BLE_GAP_EVT_DISCONNECTED:
            on_disconnect(ble_cus_t * p_cus, ble_evt_t * p_ble_evt);
            break;

        default:
            // No implementation needed.
            break;
    }
}
```

The last thing we have to do is make sure that the application dispatches the SoftDevice events to our ble_cus_on_ble_evt event handler function. This is done by adding the function call to the ble_evt_dispatch() function in main.c

```c
static void ble_evt_dispatch(ble_evt_t * p_ble_evt)
{
    /** The Connection state module has to be fed BLE events in order to function correctly
     * Remember to call ble_conn_state_on_ble_evt before calling any ble_conns_state_* functions. */
    ble_conn_state_on_ble_evt(p_ble_evt);
    pm_on_ble_evt(p_ble_evt);
    ble_conn_params_on_ble_evt(p_ble_evt);
    bsp_btn_ble_on_ble_evt(p_ble_evt);
    on_ble_evt(p_ble_evt);
    ble_advertising_on_ble_evt(p_ble_evt);
    nrf_ble_gatt_on_ble_evt(&m_gatt, p_ble_evt);
   
    //YOUR_JOB add calls to _on_ble_evt functions from each service your application is using
    ble_cus_on_ble_evt(&m_cus, p_ble_evt);
    
}
```

Great, we're now handling 
### Step X - Handling the Write event from the SoftDevice.

 but we want to be able to write to the characteristic and perform a specific task based on the value that was written to the characteristic, e.g. turn on a LED