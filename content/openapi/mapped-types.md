### 映射类型

当你构建像 **CRUD**（创建/读取/更新/删除）这样的功能时，通常需要在基础实体类型上构建变体。Nest 提供了多个实用函数来进行类型转换，以使此任务更加便捷。

#### Partial

在构建输入验证类型（也称为 DTO）时，通常需要为同一类型构建 **创建** 和 **更新** 的变体。例如，**创建** 变体可能要求所有字段都是必填的，而 **更新** 变体可能允许所有字段是可选的。

Nest 提供了 `PartialType()` 实用函数来简化此任务并减少样板代码。

`PartialType()` 函数返回一个类型（类），该类型的所有属性都是输入类型的可选属性。例如，假设我们有如下的 **创建** 类型：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

默认情况下，所有这些字段都是必填的。要创建一个具有相同字段但每个字段都可选的类型，可以使用 `PartialType()`，并将类引用 (`CreateCatDto`) 作为参数传入：

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **提示** `PartialType()` 函数从 `@nestjs/swagger` 包中导入。

#### Pick

`PickType()` 函数通过从输入类型中选取一组属性来构造一个新类型（类）。例如，假设我们有如下类型：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

我们可以使用 `PickType()` 实用函数从该类中选取一组属性：

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **提示** `PickType()` 函数从 `@nestjs/swagger` 包中导入。

#### Omit

`OmitType()` 函数通过选取输入类型的所有属性，然后移除一组特定的键来构造一个类型。例如，假设我们有如下类型：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

我们可以生成一个派生类型，它包含除 `name` 之外的所有属性，如下所示。在这个结构中，`OmitType` 的第二个参数是一个属性名的数组：

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **提示** `OmitType()` 函数从 `@nestjs/swagger` 包中导入。

#### Intersection

`IntersectionType()` 函数将两个类型合并为一个新类型（类）。例如，假设我们有如下两个类型：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  breed: string;
}

export class AdditionalCatInfo {
  @ApiProperty()
  color: string;
}
```

我们可以生成一个新类型，它包含两个类型中的所有属性：

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **提示** `IntersectionType()` 函数从 `@nestjs/swagger` 包中导入。

#### 组合使用

类型映射的实用函数是可以组合使用的。例如，下面的代码将生成一个类型（类），它具有 `CreateCatDto` 类型的所有属性，除了 `name`，并且这些属性将被设置为可选：

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```