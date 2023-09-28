# Example project using safe-protocol


# safe protocol

[safe protocol whitepaper](https://github.com/safe-global/safe-core-protocol-specs)
[safe core protocol github repo](https://github.com/safe-global/safe-core-protocol)
[safe core demo](https://github.com/5afe/safe-core-protocol-demo/)

# safe protocol overviews

## Contract structures
### SafeProtocolManager

```mermaid
classDiagram
    Ownable2Step - openzeppelin <|-- RegistryManager
    OnlyAccountCallable <|-- RegistryManager

    RegistryManager <|-- HooksManager

    RegistryManager <|-- SafeProtocolManager
    HooksManager <|-- SafeProtocolManager
    FunctionHandlerManager <|-- SafeProtocolManager

    RegistryManager <|-- FunctionHandlerManager

    IERC165 <|-- SafeProtocolManager
    ISafeProtocolManager <|-- SafeProtocolManager

    HooksManager: mapping[address => address] public enabledHooks
    HooksManager: mapping[address => TempHooksInfo] public tempHooksData
    HooksManager: +getEnabledHooks(address account)- external
    HooksManager: +setHooks(address hooks) external-external

    note for FunctionHandlerManager "fallback handler function checks \n if an account (msg.sender) has \n a function handler enabled.\n If enabled, calls handle function\n and returns the result back"
    FunctionHandlerManager: mapping[address => mapping[bytes4 => address]] public functionHandlers
    FunctionHandlerManager: +getFunctionHandler(address account, bytes4 selector)- external
    FunctionHandlerManager: +setFunctionHandler(bytes4 selector, address functionHandler)- external
    FunctionHandlerManager: +fallback()- external


    RegistryManager: address public registry
    RegistryManager: +setRegistry(address newRegistry)- external

    SafeProtocolManager: mapping[address => mapping[address => PluginAccessInfo]] public enabledPlugins
    SafeProtocolManager: +executeTransaction(address account, SafeTransaction transaction)- external
    SafeProtocolManager: +executeRootAccess(address account, SafeRootAccess rootAccess)-external
    SafeProtocolManager: +enablePlugin(address plugin, uint8 permission)-external
    SafeProtocolManager: +disablePlugin(address prevPlugin, address plugin)-external
    SafeProtocolManager: +getPluginInfo(address account, address plugin)-external
    SafeProtocolManager: +getPluginsPagenated(address start, uint256 pageSize, address account)-external
    SafeProtocolManager: +checkTransaction(address to, ...)-external
    SafeProtocolManager: +checkAfterExecution(bytes32, bool success)-external
    SafeProtocolManager: +checkModuleTransaction( address to, uint256 value, bytes memory data, Enum.Operation operation, address module)-external
    SafeProtocolManager: +supportsInterface(bytes4 interfaceId)-external 
    SafeProtocolManager: +isPluginEnabled(address account, address plugin)-public

```

### SafeProtocolRegistry
```mermaid
classDiagram

    IERC165 <|-- ISafeProtocolRegistry
    ISafeProtocolRegistry <|-- SafeProtocolRegistry
    Ownable2Step - openzeppelin <|-- SafeProtocolRegistry
    
    SafeProtocolRegistry: mapping[address => ModuleInfo] public listedModules
    SafeProtocolRegistry: +check(address module, bytes32 data) external
    SafeProtocolRegistry: +addModule(address module, uint8 moduleTypes) external
    SafeProtocolRegistry: +flagModule(address module) external

```

### Plugin
```mermaid
classDiagram
    IERC165 <|-- ISafeProtocolPlugin
    ISafeProtocolPlugin <|-- PluginInstance
    
    note for ISafeProtocolPlugin "for further details about metadataProvider \n https://github.com/safe-global/safe-core-protocol-specs/blob/main/metadata/README.md"
    ISafeProtocolPlugin: +name() external
    ISafeProtocolPlugin: +version() external
    ISafeProtocolPlugin: +metadataProvider() external
    ISafeProtocolPlugin: +requiresPermissions() external

```

## Contract interructions
### Enabling Plugin

```mermaid
sequenceDiagram
    participant User
    participant Account as Account(safe)
    participant Registory
    participant Manager

    User->>User: Deploy Registory(owner)
    User->>User: Deploy Manager(owner, registory address) 
    User->>User: Deploy Safe(...manager address for fallback handler?)
    User->>User: Deploy Plugin
    User->>Registory: addModule(plugin address, MODULE_TYPE_PLUGIN)
    User->>Account: executeTransaction("enablePlugin[plugin address, PLUGIN_PERMISSION_EXECUTE_CALL]"...)
    Account->>Account: call falback handler (manager)
    Account->>Manager: enablePlugin(plugin address, PLUGIN_PERMISSION_EXECUTE_CALL)
```
manager.getPuginInfo(account address, plugin address) should return PLUGIN_PERMISSION_EXECUTE_CALL and SENTINEL_MODULES

### Execute trnsaction through Plugin

```mermaid
sequenceDiagram
    participant User
    participant Plugin
    participant Manager
    participant Account as Account(safe)

    User->>Plugin: callFunction(ISafeProtocolManager manager, safe, ...)
    Plugin->>Plugin: create SafeProtocolAction
    Plugin->>Plugin: create SafeRootAccess
    Plugin->>Manager: executeRootAccess(safe, SafeRootAccess)
    Manager->>Manager: check if hook is enabled
    Manager->>Manager: call hook.preCheckRootAccess if hook is enabled
    Manager->>Manager: check permission of safe (PLUGIN_PERMISSION_EXECUTE_DELEGATECALL)
    Manager->>Account: executeTransactionFromModuleReturnData using SafeRootAccess.action
    Manager->>Manager: call hook.postCheck if hook is enabled
    Manager->>Manager: emit event (RootAccessActionExecuted or RootAccessActionExecutionFailed)

```

### Execute transaction through Hook
### Execute transaction through FunctionHandler