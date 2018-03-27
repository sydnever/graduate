# 代码实现

## 概要说明

所有和Web相关的代码都放在了UI文件夹下

使用React库实现数据展示层，nodejs实现轻量级的WebServer。

使用go语言专门的方法，将WebServer嵌合到GreatDB的二进制文件中。

## 详细文件说明

### ui.go

将nodejs代码整合编译到最终的二进制文件中。

```go
var indexHTML = []byte(fmt.Sprintf(`<!DOCTYPE html>
<title>GreatDB</title>
Binary built without web UI.
<hr>
<em>%s</em>`, build.GetInfo().Short()))

// Asset加载并返回指定名字的Asset。当Asset无法加载时会抛出异常
var Asset = func(name string) ([]byte, error) {
	if name == "index.html" {
		return indexHTML, nil
	}
	return nil, os.ErrNotExist
}

// AssetDir 返回嵌入式文件系统中一个目录下的文件名。
var AssetDir = func(name string) ([]string, error) {
	if name == "" {
		return []string{"index.html"}, nil
	}
	return nil, os.ErrNotExist
}

// AssetInfo 加载并返回指定名字asset的元数据。
var AssetInfo func(name string) (os.FileInfo, error)
```

### fonts目录
存放字体文件。

### styl目录
存放css文件。

### assets目录。
存放svg，gif等格式的图片资源文件。

### opt目录
存放Web显示层所依赖的外部库。

### src目录

#### hacks目录
rc-progress.d.ts

```ts
declare module "rc-progress" {
  export interface LineProps {
    strokeWidth?: number;
    trailWidth?: number;
    className?: string;
    percent?: number;
  }
  export class Line extends React.Component<LineProps, {}> {
  }
}
```
导出一个带有LineProps接口的Line类。

#### interfaces目录
各种接口的声明。

action.d.ts

```ts
import { Action } from "redux";


export interface PayloadAction<T> extends Action {
  payload: T;
}

interface WithRequest<T, R> {
  data?: T;
  request: R;
}
```

assets.d.ts

```ts
declare module "assets/*" {
    var _: string;
    export default _;
}

declare module "!!raw-loader!*" {
    var _: string;
    export default _;
}
```

layout.d.ts
```ts
import React from "react";
import { RouterProps } from "react-router";

export interface TitledComponent {
  title(routeProps: RouterProps): React.ReactElement<any>;
}
```
#### js目录

主要是javascript的protobuf的定义与实现，用于获取每个GreatDB节点上需要监控的数据。

object-assign.d.ts
```ts
interface ObjectConstructor {
    assign(target: any, ...sources: any[]): any;
}
```

object-assign.js
```js
if (typeof Object.assign != 'function') {
  (function () {
    Object.assign = function (target) {
      'use strict';
      if (target === undefined || target === null) {
        throw new TypeError('Cannot convert undefined or null to object');
      }

      var output = Object(target);
      for (var index = 1; index < arguments.length; index++) {
        var source = arguments[index];
        if (source !== undefined && source !== null) {
          for (var nextKey in source) {
            if (source.hasOwnProperty(nextKey)) {
              output[nextKey] = source[nextKey];
            }
          }
        }
      }
      return output;
    };
  })();
}
```
protos.d.ts

Web服务器用于gRPC通讯的protobuf的定义，涵盖了所有获取所需信息的操作，总计35509行。

proto.js

所有跟protobuf相关的实现，总计69277行。

#### redux目录

和React的redux机制相关，包含接近三十个文件。

### routes目录
与React路由相关。

visualization.tsx
```tsx
import React from "react";
import { Route, IndexRedirect } from "react-router";

import { NodesOverview } from "src/views/cluster/containers/nodesOverview";
import ClusterOverview from "src/views/cluster/containers/clusterOverview";

class NodesWrapper extends React.Component<{}, {}> {
  render() {
    return (
      <div style={{ marginTop: 12 }}>
        <NodesOverview />
      </div>
    );
  }
}

export default function(): JSX.Element {
  return (
    <Route path="overview" component={ ClusterOverview } >
      <IndexRedirect to="list" />
      <Route path="list" component={ NodesWrapper } />
    </Route>
  );
}

```
### util目录
一些零散的工具，包含近三十个文件。

### views目录

与数据可视化相关，会根据模板生成对用户可见的前端页面。

#### app目录

所有页面共用的组件，包括Sidebar的布局，NotFound页面，alterBanner和timewindow。

#### cluster目录

集群信息页面。

#### databases目录

数据库信息界面。

#### devtools

开发工具。

#### jobs

历史作业界面。

#### reports

报告页面。