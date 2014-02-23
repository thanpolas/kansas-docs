
```js
var kansas = kansasApi();

kansas.setPolicy('free', {
  maxApps: 3,
  maxTokensPerApp: 3,
  requestsLimit: 5000,
  limitPeriod: 'month',
});
```