# 修复Swagger生成Dart代码中空参数名问题

## 问题描述

当Swagger JSON中的parameters数组包含只有`"in": "query"`而没有`name`字段的对象时，会生成错误的Dart代码。

### 问题示例

**Swagger JSON:**
```json
{
  "parameters": [
    {
      "name": "outTradeNo",
      "in": "query",
      "description": "订单id",
      "required": true,
      "schema": {
        "type": "string"
      }
    },
    {
      "in": "query"
    }
  ]
}
```

**修复前生成的错误代码:**
```dart
///查看订单信息
///@param outTradeNo 订单id
///@param
@Get(path: '/api/recharge/record/get/record/info')
Future<chopper.Response<BusiResultOrderVO>>
    _apiRechargeRecordGetRecordInfoGet({
  @Query('outTradeNo') required String? outTradeNo,
  @Query('') Object? $,  // ❌ 这里生成了错误的参数
});
```

## 解决方案

在 `lib/src/code_generators/swagger_requests_generator.dart` 文件的 `_getAllParameters` 方法中添加了对参数名的验证：

### 修复的代码

```dart
final result = parameters
    .where((swaggerParameter) =>
        ignoreHeaders ? swaggerParameter.inParameter != kHeader : true)
    .where((swaggerParameter) => swaggerParameter.inParameter != kCookie)
    .where((swaggerParameter) => swaggerParameter.inParameter.isNotEmpty)
    .where((swaggerParameter) => swaggerParameter.name.isNotEmpty)  // ✅ 新增：过滤空name的参数
    .map(
      (swaggerParameter) => Parameter(
        // ... 其余代码保持不变
      ),
    )
    .toList();
```

### 修复后生成的正确代码

```dart
///查看订单信息
///@param outTradeNo 订单id
@Get(path: '/api/recharge/record/get/record/info')
Future<chopper.Response<BusiResultOrderVO>>
    _apiRechargeRecordGetRecordInfoGet({
  @Query('outTradeNo') required String? outTradeNo,  // ✅ 只生成有效参数
});
```

## 验证

添加了专门的测试来验证修复：

```dart
test('Should not generate parameters with empty names', () {
  final result = SwaggerRequestsGenerator(GeneratorOptions(
    inputFolder: '',
    outputFolder: '',
    ignoreHeaders: false,
  )).generate(
    swaggerRoot: root,
    className: 'CarsService',
    fileName: 'cars_service',
    allEnums: [],
  );

  // 验证不会生成无效的参数
  expect(result, isNot(contains("@Query('') Object? \$")));
  expect(result, isNot(contains("@Query('')")));
});
```

## 影响范围

- ✅ 修复了无效参数生成的问题
- ✅ 保持了现有功能的完整性
- ✅ 所有现有测试仍然通过
- ✅ 不会影响正常的参数处理逻辑

这个修复确保了只有具有有效名称的参数才会被处理和生成到最终的Dart代码中。 