<!-- version -->
# 2.0.0 API Reference
<!-- versionstop -->

<!-- toc -->

- [Engine](#engine)
  - [`new Engine([options])`](#new-engineoptions)
  - [`execute([options], [callback])`](#executeoptions-callback)
    - [Execution `variables`](#execution-variables)
    - [Execution `services`](#execution-services)
    - [Execution `listener`](#execution-listener)
  - [`getState()`](#getstate)
  - [`stop()`](#stop)
  - [`resume(state, [options], [callback])`](#resumestate-options-callback)
  - [Engine events](#engine-events)
  - [Activity events](#activity-events)
    - [Element events](#element-events)
    - [Sequence flow events](#sequence-flow-events)
  - [Expressions](#expressions)
  - [Task loop](#task-loop)
    - [Cardinality loop](#cardinality-loop)
    - [Conditional loop](#conditional-loop)
    - [Collection loop](#collection-loop)
- [Transformer](#transformer)
  - [`transform(sourceXml, options, callback)`](#transformsourcexml-options-callback)
- [Examples](#examples)
  - [Execute](#execute)
  - [Listen for events](#listen-for-events)
  - [Exclusive gateway](#exclusive-gateway)
  - [Script task](#script-task)
  - [User task](#user-task)
  - [Service task](#service-task)
  - [Sequence flow with expression](#sequence-flow-with-expression)
  - [Task loop over collection](#task-loop-over-collection)

<!-- tocstop -->

## Engine

### `new Engine([options])`

Creates a new Engine object where:

- `options`: Optional object
  - `source`: Bpmn definition source as String or Buffer
  - `context`: Optional parsed moddle context object
  - `name`: Optional name of engine,
  - `moddleOptions`: Optional moddle parse options

Options `source` and `context` are mutually exclusive.

Moddle options can be used if an extension is required when parsing BPMN-source. The object will be passed on to the constructor of `bpmn-moddle`. See [camunda-bpmn-moddle][1] for example.

```javascript
const Bpmn = require('bpmn-engine');
const fs = require('fs');

const engine = new Bpmn.Engine({
  source: fs.readFileSync('./test/resources/mother-of-all.bpmn'),
  moddleOptions: {
    camunda: require('camunda-bpmn-moddle/resources/camunda')
  }
});
```

### `execute([options], [callback])`

Execute process with:

- `options`: Optional object
  - [`listener`](#execution-listener): an `EventEmitter` object
  - [`variables`](#execution-variables): Optional object with instance variables
  - [`services`](#execution-services): Optional object with service definitions
- `callback`: optional callback
  - `err`: Error if any
  - `instance`: Main process instance
  - `siblings`: List of all processes

```javascript
'use strict';

const Bpmn = require('bpmn-engine');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <dataObjectReference id="inputFromUserRef" dataObjectRef="inputFromUser" />
    <dataObject id="inputFromUser" />
    <startEvent id="theStart" />
    <userTask id="userTask">
      <ioSpecification id="inputSpec">
        <dataOutput id="userInput" />
      </ioSpecification>
      <dataOutputAssociation id="associatedWith" sourceRef="userInput" targetRef="inputFromUserRef" />
    </userTask>
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

engine.execute((err, instance) => {
  if (err) throw err;
  instance.once('end', () => {
    console.log('completed')
  });
});
```

#### Execution `variables`

Execution variables are passed as the first argument to `#execute`. To be able to distinguish between variables and services the safest way is to pass the variables as any other name than `services`.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const fs = require('fs');

const engine = new Bpmn.Engine({
  source: fs.readFileSync('./test/resources/simple-task.bpmn')
});

const variables = {
  input: 1
};

engine.execute({
  variables: variables
}, (err, instance) => {
  if (err) throw err;
  instance.once('end', () => {
    console.log('completed')
  });
});
```

#### Execution `services`

A service is a module used by e.g. a script tasks or a condition where:

```javascript
const options = {
  services: {
    get: {
      module: 'request',
      type: 'require',
      fnName: 'get'
    },
    checkState: (message) => {
      return message.variables.statusCode === 200;
    }
  }
};
```

- `name`: Exposed name in the engine
  - `module`: Module or global name
  - `type`: Optional type, supported types are `require` and `global`, defaults to `require`
  - `fnName`: Optional module function name

If the process will be stopped and resumed the module structure is to prefer. But it is also possible to send in functions, aware that they will not be serialized when getting state.

The module name can be a npm module, local module or a global reference. If a local module is used the path must be relative from execution path, i.e. `process.cwd()`.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
  <startEvent id="theStart" />
  <scriptTask id="scriptTask" scriptFormat="Javascript">
    <script>
      <![CDATA[
        const request = services.request;

        const self = this;

        request.get('http://example.com/test', (err, resp, body) => {
          if (err) return next(err);
          self.variables.scriptTaskCompleted = true;
          next(null, {result: body});
        })
      ]]>
    </script>
  </scriptTask>
  <endEvent id="theEnd" />
  <sequenceFlow id="flow1" sourceRef="theStart" targetRef="scriptTask" />
  <sequenceFlow id="flow2" sourceRef="scriptTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

const services = {
  request: {
    module: 'request'
  },
  put: {
    module: 'request',
    fnName: 'put'
  },
  require: {
    module: 'require',
    type: 'global'
  },
  serviceFn: {
    module: './test/helpers/testHelpers',
    fnName: 'serviceFn'
  }
};

engine.execute({
  services: services
}, (err, instance) => {
  if (err) throw err;
  instance.once('end', () => {
    console.log('completed')
  });
});
```

#### Execution `listener`

An `EventEmitter` object with listeners. [Event names](#activity-events) are composed by activity event name and activity id, e.g. `wait-userTask`.

Event arguments are:

- `activity`: The activity instance
- `instance`: The running process instance

### `getState()`

Get state of a running execution.

The saved state will include the following content:

- `source`: Buffered representation of the definition
- `sourceHash`: Calculated md5 hash of the executing definition
- `context`: Moddle context
- `moddleOptions`: Engine moddleOptions
- `processes`: Object with processes with id as key
  - `variables`: Execution variables
  - `services`: Execution services
  - `children`: List of child states
    - `entered`: Boolean indicating if the child is currently executing

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;
const fs = require('fs');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <startEvent id="theStart" />
    <userTask id="userTask" />
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

const listener = new EventEmitter();

let state;
listener.once('wait-userTask', (activity) => {
  state = engine.getState();
  fs.writeFileSync('./tmp/some-random-id.json', JSON.stringify(state, null, 2));
  console.log(JSON.stringify(state, null, 2));
});

engine.execute({
  listener: listener
}, (err, execution) => {
  if (err) throw err;
});
```

### `stop()`

Stop execution. The instance is terminated.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <startEvent id="theStart" />
    <userTask id="userTask" />
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});
const listener = new EventEmitter();

let state;
listener.once('wait-userTask', (activity) => {
  engine.stop();
  state = engine.getState();
});

engine.execute({
  variables: {
    executionId: 'some-random-id'
  },
  listener: listener
}, (err, execution) => {
  if (err) throw err;
});
```

### `resume(state, [options], [callback])`

Execute engine `#resume` function with previously saved state.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

// Retrieve saved state
const state = db.getSavedState('some-random-id', (err, state) => {
  if (err) return console.log(err.message);

  const engine = new Bpmn.Engine({
    context: state.context
  });

  engine.on('end', () => {
    console.log('resumed instance completed');
  });

  engine.resume(state, (err, instance) => {
    if (err) throw err;
  });
});
```

### Engine events

Engine emits the following events:

- `end`: Execution has completed or was stopped.

### Activity events

Each activity and flow emits events when changing state.

#### Element events

- `enter`: An element is entered
- `start`: An element is started
- `wait`: An user task waits for signal
- `end`: A task has ended successfully
- `cancel`: An element execution was canceled
- `leave`: The execution left the element
- `error`: An error was emitted (unless a bound error event is in place)

#### Sequence flow events

- `taken`: The sequence flow was taken
- `discarded`: The sequence flow was discarded

### Expressions

Expressions come in the form of `${<variables or services>.<property name>}`.

The following expressions are supported:

- `${variables.input}` - resolves to the variable input
- `${variables.input[0]}` - resolves to first item of the variable input array
- `${variables.input[spaced name]}` - resolves to the variable input object property `spaced name`

- `${services.getInput}` - return the service function `getInput`
- `${services.getInput()}` - executes the service function `getInput` with the argument `{services, variables}`
- `${services.isBelow(variables.input,2)}` - executes the service function `isBelow` with result of `variable.input` value and 2

Expressions are supported in the following elements:
- MultiInstanceLoopCharacteristics
  - `camunda:collection`: variable collection, e.g. `${variables.list}`
- ServiceTask
  - `camunda:expression` element value. moddleOptions [`require('camunda-bpmn-moddle/resources/camunda')`][1] must be used.
- SequenceFlow
  - `conditionExpression` element value
- TimerEvent
  - `timeDuration` element value

Expressions in expressions is **not** supported.

### Task loop

Task loops can made based conditions, cardinality, or a collection.

#### Cardinality loop

Loop a fixed number of times or until number of iterations match cardinality. The condition body can be a script or an expression.

```xml
<bpmn:loopCardinality xsi:type="bpmn:tFormalExpression">${variables.list.length}</bpmn:loopCardinality>
```

#### Conditional loop

Loop until condition is met. The condition body can be a script or an expression.

```xml
<completionCondition xsi:type="tFormalExpression">\${services.condition(variables.input)}</completionCondition>
```

#### Collection loop

Loop all items in a list.

```xml
<bpmn:multiInstanceLoopCharacteristics isSequential="true" camunda:collection="\${variables.list}" />
```

For `bpmn-moddle` to read the `camunda:collection` namespaced attribute, the engine must be instantiated with moddle options referring [`camunda-bpmn-moddle/resources/camunda`][1].

## Transformer

Basically a wrapper around [`bpmn-moddle.fromXml`][2].

### `transform(sourceXml, options, callback)`

- `sourceXml`: String with bpmn definition source
- `options`: Options to pass to [`bpmn-moddle`][2]
- `callback`: callback
  - `err`: Occasional error
  - `definition`: Bpmn definition
  - `context`: Bpmn moddle context

```javascript
const Bpmn = require('bpmn-engine');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <startEvent id="theStart" />
    <userTask id="userTask" />
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

Bpmn.Transformer.transform(processXml, {
  camunda: require('camunda-bpmn-moddle/resources/camunda')
}, (err, def, moddleContext) => {
  const engine = new Bpmn.Engine({
    context: moddleContext
  });

  engine.execute({
    variables: {
      shortid: '42'
    }
  }, (err, instance) => {
    console.log('Process instance started with id', instance.context.variables.shortid);
  });
});
```

## Examples

### Execute
```javascript
const Bpmn = require('bpmn-engine');
const shortid = require('shortid');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
  <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess2" isExecutable="true">
    <startEvent id="theStart" />
    <exclusiveGateway id="decision" default="flow2" />
    <endEvent id="end1" />
    <endEvent id="end2" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="decision" />
    <sequenceFlow id="flow2" sourceRef="decision" targetRef="end1" />
    <sequenceFlow id="flow3" sourceRef="decision" targetRef="end2">
      <conditionExpression>true</conditionExpression>
    </sequenceFlow>
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

engine.execute({
  variables: {
    shortid: shortid.generate()
  }
}, (err, instance) => {
  console.log('Process instance started with id', instance.context.variables.shortid);
});
```

### Listen for events

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <dataObjectReference id="inputFromUserRef" dataObjectRef="inputFromUser" />
    <dataObject id="inputFromUser" />
    <startEvent id="theStart" />
    <userTask id="userTask">
      <ioSpecification id="inputSpec">
        <dataOutput id="userInput" />
      </ioSpecification>
      <dataOutputAssociation id="associatedWith" sourceRef="userInput" targetRef="inputFromUserRef" />
    </userTask>
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});
const listener = new EventEmitter();

listener.once('wait-userTask', (task) => {
  task.signal({
    sirname: 'von Rosen'
  });
});

engine.execute({
  listener: listener
}, (err, instance) => {
  if (err) throw err;

  instance.once('end', () => {
    console.log(instance.variables)
    console.log(`User sirname is ${instance.variables.inputFromUser.sirname}`);
  });
});
```

### Exclusive gateway

An exclusive gateway will receive the available process variables as `this.variables`.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <startEvent id="start" />
    <exclusiveGateway id="decision" />
    <endEvent id="end1" />
    <endEvent id="end2" />
    <sequenceFlow id="flow1" sourceRef="start" targetRef="decision" />
    <sequenceFlow id="flow2" sourceRef="decision" targetRef="end1">
      <conditionExpression xsi:type="tFormalExpression" language="JavaScript"><![CDATA[
      this.variables.input <= 50
      ]]></conditionExpression>
    </sequenceFlow>
    <sequenceFlow id="flow3" sourceRef="decision" targetRef="end2">
      <conditionExpression xsi:type="tFormalExpression" language="JavaScript"><![CDATA[
      this.variables.input > 50
      ]]></conditionExpression>
    </sequenceFlow>
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

engine.execute({
  variables: {
    input: 51
  }
}, (err, instance) => {
  if (err) throw err;

  instance.once('end', () => {
    if (instance.getChildActivityById('end1').taken) throw new Error('<end1> was not supposed to be taken, check your input');
    console.log('TAKEN end2', instance.getChildActivityById('end2').taken);
  });
});
```

### Script task

A script task will receive the data available on the process instance. So if `request` or another module is needed it has to be passed when starting the process. The script task also has a callback called `next` that has to be called for the task to complete.

The `next` callback takes the following arguments:
- `err`: Occasional error
- `result`: The result of the script

```javascript
'use strict';

const Bpmn = require('bpmn-engine');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
  <startEvent id="theStart" />
  <scriptTask id="scriptTask" scriptFormat="Javascript">
    <script>
      <![CDATA[
        const request = services.request;

        const self = this;

        request.get('http://example.com/test', (err, resp, body) => {
          if (err) return next(err);
          self.variables.scriptTaskCompleted = true;
          next(null, {result: body});
        })
      ]]>
    </script>
  </scriptTask>
  <endEvent id="theEnd" />
  <sequenceFlow id="flow1" sourceRef="theStart" targetRef="scriptTask" />
  <sequenceFlow id="flow2" sourceRef="scriptTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

engine.execute({
  variables: {
    scriptTaskCompleted: false
  },
  services: {
    request: {
      module: 'request'
    }
  }
}, (err, execution) => {
  if (err) throw err;

  execution.once('end', () => {
    console.log('Script task modification:', execution.variables.scriptTaskCompleted);
    console.log('Script task output:', execution.variables.taskInput.scriptTask.result);
  });
});
```

### User task

User tasks waits for signal to complete. The signal function can be called on the emitted event or the executing instance.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <dataObjectReference id="inputFromUserRef" dataObjectRef="inputFromUser" />
    <dataObject id="inputFromUser" />
    <startEvent id="theStart" />
    <userTask id="userTask">
      <ioSpecification id="inputSpec">
        <dataOutput id="userInput" />
      </ioSpecification>
      <dataOutputAssociation id="associatedWith" sourceRef="userInput" targetRef="inputFromUserRef" />
    </userTask>
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});
const listener = new EventEmitter();

listener.once('wait-userTask', (child, instance) => {
  instance.signal(child.activity.id, {
    sirname: 'von Rosen'
  });
});

engine.execute({
  listener: listener
}, (err, instance) => {
  if (err) throw err;

  instance.once('end', () => {
    console.log(`User sirname is ${instance.variables.inputFromUser.sirname}`);
  });
});
```

Since, Imho, the data flow in bpmn2 is overcomplex the input is stored as `taskInput` with id if data associations dont´t exist.

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
    <startEvent id="theStart" />
    <userTask id="userTask" />
    <endEvent id="theEnd" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="userTask" />
    <sequenceFlow id="flow2" sourceRef="userTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

const listener = new EventEmitter();

listener.once('wait-userTask', (child, instance) => {
  instance.signal(child.activity.id, {
    sirname: 'von Rosen'
  });
});

engine.execute({
  listener: listener
}, (err, instance) => {
  if (err) throw err;

  instance.once('end', () => {
    console.log(`User sirname is ${instance.variables.taskInput.userTask.sirname}`);
  });
});
```

### Service task

A service task will receive the data available on the process instance. The signature of the service function is:

- `message`:
  - `variables`: Instance variables
  - `services`: All instance services
- `callback`:
  - `err`: Occasional error
  - `result`: The results of the service call arguments without the first argument (error)

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const request = require('request');

const services = require('../test/helpers/testHelpers');
services.getRequest = (message, callback) => {

  request.get({
    uri: message.variables.apiPath,
    json: true
  }, (err, resp, body) => {
    if (err) return callback(err);
    return callback(null, {
      statusCode: resp.statusCode,
      data: body
    });
  });
};

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:camunda="http://camunda.org/schema/1.0/bpmn" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess" isExecutable="true">
  <startEvent id="theStart" />
  <serviceTask id="serviceTask">
    <extensionElements>
      <camunda:properties>
        <camunda:property name="service" value="getRequest" />
      </camunda:properties>
    </extensionElements>
  </serviceTask>
  <endEvent id="theEnd" />
  <sequenceFlow id="flow1" sourceRef="theStart" targetRef="serviceTask" />
  <sequenceFlow id="flow2" sourceRef="serviceTask" targetRef="theEnd" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml
});

engine.execute({
  variables: {
    apiPath: 'http://example.com/test'
  },
  services: {
    getRequest: {
      module: './test/helpers/testHelpers',
      fnName: 'getRequest'
    }
  }
}, (err, execution) => {
  if (err) throw err;

  execution.once('end', () => {
    console.log('Service task output:', execution.variables.taskInput.serviceTask.result);
  });
});
```

or if arguments must be passed, then `inputParameter` `arguments` must be defined. The result is an array with arguments from the service callback where first error argument is omitted.

```javascript
const Bpmn = require('bpmn-engine');

const processXml = `
  <?xml version="1.0" encoding="UTF-8"?>
  <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:camunda="http://camunda.org/schema/1.0/bpmn" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <process id="Process_1" isExecutable="true">
      <startEvent id="start">
        <outgoing>flow1</outgoing>
      </startEvent>
      <sequenceFlow id="flow1" sourceRef="start" targetRef="serviceTask" />
      <endEvent id="end">
        <incoming>flow2</incoming>
      </endEvent>
      <sequenceFlow id="flow2" sourceRef="serviceTask" targetRef="end" />
      <serviceTask id="serviceTask" name="Get">
        <extensionElements>
          <camunda:inputOutput>
            <camunda:inputParameter name="uri">
              <camunda:script scriptFormat="JavaScript">variables.apiPath</camunda:script>
            </camunda:inputParameter>
            <camunda:outputParameter name="result">
              <camunda:script scriptFormat="JavaScript"><![CDATA[
'use strict';
var result = {
  statusCode: result[0].statusCode,
  body: result[0].statusCode === 200 ? JSON.parse(result[1]) : undefined
};
result;
              ]]>
              </camunda:script>
            </camunda:outputParameter>
          </camunda:inputOutput>
          <camunda:properties>
            <camunda:property name="service" value="getRequest" />
          </camunda:properties>
        </extensionElements>
        <incoming>flow1</incoming>
        <outgoing>flow2</outgoing>
      </serviceTask>
    </process>
  `;

const engine = new Bpmn.Engine({
  source: processXml
});
engine.execute({
  variables: {
    apiPath: 'http://example.com/test'
  },
  services: {
    getRequest: {
      module: 'request',
      fnName: 'get'
    }
  }
}, (err, execution) => {
  if (err) throw err;
  execution.once('end', () => {
    console.log(execution.variables)
    console.log('Script task output:', execution.variables.result);
  });
});
```

In the above example `request.get` will be called with `variables.apiPath`. The result is passed through `outputParameter` `result`.

Expressions can also be used if camunda extension `moddleOptions` are passed to engine.

```javascript
const Bpmn = require('bpmn-engine');

const processXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:camunda="http://camunda.org/schema/1.0/bpmn">
  <process id="theProcess" isExecutable="true">
    <serviceTask id="serviceTask" name="Get" camunda:expression="\${services.getService()}" camunda:resultVariable="output" />
  </process>
</definitions>`;

const engine = new Bpmn.Engine({
  source: processXml,
  moddleOptions: {
    camunda: require('camunda-bpmn-moddle/resources/camunda')
  }
});

engine.execute({
  services: {
    getService: () => {
      return (executionContext, callback) => {
        callback(null, executionContext.variables.input);
      };
    }
  },
  variables: {
    input: 1
  }
}, (err, instance) => {
  if (err) return done(err);
  instance.once('end', () => {
    console.log(instance.variables.taskInput.serviceTask.output);
  });
});
```

### Sequence flow with expression

```javascript
'use strict';

const Bpmn = require('bpmn-engine');
const EventEmitter = require('events').EventEmitter;

const sourceXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions id="testProcess" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <process id="theProcess1" isExecutable="true">
    <startEvent id="theStart" />
    <exclusiveGateway id="decision" default="flow2" />
    <endEvent id="end1" />
    <endEvent id="end2" />
    <sequenceFlow id="flow1" sourceRef="theStart" targetRef="decision" />
    <sequenceFlow id="flow2" sourceRef="decision" targetRef="end1" />
    <sequenceFlow id="flow3withExpression" sourceRef="decision" targetRef="end2">
      <conditionExpression xsi:type="tFormalExpression">\${services.isBelow(variables.input,2)}</conditionExpression>
    </sequenceFlow>
  </process>
</definitions>
`;

const engine = new Bpmn.Engine({
  source: sourceXml
});
const listener = new EventEmitter();
listener.on('taken-flow3withExpression', (flow) => {
  throw new Error(`<${flow.id}> should not have been taken`);
});
engine.execute({
  listener: listener,
  services: {
    isBelow: (input, test) => {
      return input < Number(test);
    }
  },
  variables: {
    input: 2
  }
}, (err, instance) => {
  if (err) return done(err);
  instance.once('end', () => {
    console.log('WOHO!');
  });
});

```

### Task loop over collection

```javascript
'use strict';

const Bpmn = require('bpmn-engine');

const sourceXml = `
<?xml version="1.0" encoding="UTF-8"?>
<definitions id= "Definitions_1" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" targetNamespace="http://bpmn.io/schema/bpmn">
  <process id="Process_1" isExecutable="true">
    <serviceTask id="recurring" name="Each item">
      <multiInstanceLoopCharacteristics isSequential="true" camunda:collection="\${variables.input}" />
      <extensionElements>
        <camunda:inputOutput>
          <camunda:outputParameter name="sum">\${result[0]}</camunda:outputParameter>
        </camunda:inputOutput>
        <camunda:connector>
          <camunda:connectorId>loop</camunda:connectorId>
        </camunda:connector>
      </extensionElements>
    </serviceTask>
    <boundaryEvent id="errorEvent" attachedToRef="recurring">
      <errorEventDefinition />
    </boundaryEvent>
  </process>
</definitions>
`;

const engine = new Bpmn.Engine({
  source: sourceXml,
  moddleOptions: {
    camunda: require('camunda-bpmn-moddle/resources/camunda')
  }
});

engine.execute({
  services: {
    loop: (executionContext, callback) => {
      const prevResult = executionContext.variables.sum ? executionContext.variables.sum : 0;
      const result = prevResult + executionContext.item;
      callback(null, result);
    }
  },
  variables: {
    input: [1, 2, 3, 7]
  }
}, (err, instance) => {
  if (err) return console.log(err);
  instance.once('end', () => {
    console.log(instance.variables.sum, 'aught to be 13 blazing fast');
  });
});
```

[1]: https://www.npmjs.com/package/camunda-bpmn-moddle
[2]: https://www.npmjs.com/package/bpmn-moddle
