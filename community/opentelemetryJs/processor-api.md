# Processor API Guide

[The processor](https://github.com/open-telemetry/opentelemetry-js/blob/main/packages/opentelemetry-sdk-metrics-base/src/export/Processor.ts?rgh-link-date=2020-05-25T18%3A43%3A57Z) has two responsibilities: choosing which aggregator to choose for a metric instrument and store the last record for each metric ready to be exported.

有两个职责：为指标度量工具选择哪一个聚合器，和存储每一个导出的指标的最后一条记录。

## 为指标选择特定的聚合器

有时，您可能希望对您的指标之一使用特定的聚合器，导出最后 X 值的平均值，而不是仅导出最后一个值。

下面是一个聚合器的使用样例：

```ts
import { Aggregator } from '@opentelemetry/sdk-metrics-base';
import { hrTime } from '@opentelemetry/core';

export class AverageAggregator implements Aggregator {

  private _values: number[] = [];
  private _limit: number;

  constructor (limit?: number) {
    this._limit = limit ?? 10;
  }

  update (value: number) {
    this._values.push(value);
    if (this._values.length >= this._limit) {
      this._values = this._values.slice(0, 10);
    }
  }

  toPoint() {
    const sum =this._values.reduce((agg, value) => {
      agg += value;
      return agg;
    }, 0);
    return {
      value: this._values.length > 0 ? sum / this._values.length : 0,
      timestamp: hrTime(),
    }
  }
}
```

现在我们需要实现我们自己的处理器来配置 sdk 以使用我们的新聚合器。 为了进一步简化，我们将只扩展 `UngroupedProcessor`（这是默认设置）以避免重新实现整个 `Aggregator` 接口。

结果如下：

```ts
import {
  UngroupedProcessor,
  MetricDescriptor,
  CounterSumAggregator,
  ObserverAggregator,
  MeasureExactAggregator,
} from '@opentelemetry/sdk-metrics-base';

export class CustomProcessor extends UngroupedProcessor {
  aggregatorFor (metricDescriptor: MetricDescriptor) {
    if (metricDescriptor.name === 'requests') {
      return new AverageAggregator(10);
    }
    // this is exactly what the "UngroupedProcessor" does, we will re-use it
    // to fallback on the default behavior
    switch (metricDescriptor.metricKind) {
      case MetricKind.COUNTER:
        return new CounterSumAggregator();
      case MetricKind.OBSERVER:
        return new ObserverAggregator();
      default:
        return new MeasureExactAggregator();
    }
  }
}
```

最后，我们需要指定`MeterProvider`在创建新仪表时使用我们的`CustomProcessor`：

```ts
import {
  UngroupedProcessor,
  MetricDescriptor,
  CounterSumAggregator,
  ObserverAggregator,
  MeasureExactAggregator,
  MeterProvider,
  Aggregator,
} from '@opentelemetry/sdk-metrics-base';
import { hrTime } from '@opentelemetry/core';

export class AverageAggregator implements Aggregator {

  private _values: number[] = [];
  private _limit: number;

  constructor (limit?: number) {
    this._limit = limit ?? 10;
  }

  update (value: number) {
    this._values.push(value);
    if (this._values.length >= this._limit) {
      this._values = this._values.slice(0, 10);
    }
  }

  toPoint() {
    const sum =this._values.reduce((agg, value) => {
      agg += value;
      return agg;
    }, 0);
    return {
      value: this._values.length > 0 ? sum / this._values.length : 0,
      timestamp: hrTime(),
    }
  }
}

export class CustomProcessor extends UngroupedProcessor {
  aggregatorFor (metricDescriptor: MetricDescriptor) {
    if (metricDescriptor.name === 'requests') {
      return new AverageAggregator(10);
    }
    // this is exactly what the "UngroupedProcessor" does, we will re-use it
    // to fallback on the default behavior
    switch (metricDescriptor.metricKind) {
      case MetricKind.COUNTER:
        return new CounterSumAggregator();
      case MetricKind.OBSERVER:
        return new ObserverAggregator();
      default:
        return new MeasureExactAggregator();
    }
  }
}

const meter = new MeterProvider({
  processor: new CustomProcessor(),
  interval: 1000,
}).getMeter('example-custom-processor');

const requestsLatency = meter.createValueRecorder('requests', {
  monotonic: true,
  description: 'Average latency'
});
```
