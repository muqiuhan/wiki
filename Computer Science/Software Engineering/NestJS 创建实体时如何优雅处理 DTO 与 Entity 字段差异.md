#web #software-engineering #typescript 

例如有如下患者 Entity 和创建患者的 DTO:

```typescript
export class PatientEntity implements PatientType {
  id: number;
  name: string;
  phone: string;
  gender: number;
  birthday: Date;
  address: string;
  idCard: string;
  currentResearchStatusID: number;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export class CreatePatientRequestDto {
  @ApiProperty({ description: "患者姓名", type: String, required: true })
  name: string;

  @ApiProperty({ description: "患者电话", type: String, required: true })
  phone: string;

  @ApiProperty({ description: "患者性别", type: Number, required: true })
  gender: number;

  @ApiProperty({ description: "患者生日", type: Date, required: true })
  birthday: Date;

  @ApiProperty({ description: "患者地址", type: String, required: true })
  address: string;

  @ApiProperty({ description: "患者身份证号", type: String, required: true })
  idCard: string;
}
```

在创建患者的 controller 中，`createPatientRequestDto` 中并没有 `id` 等字段，但在 service 层面的 `createPatient` 接受的 `PatientEntity` 却需要这么一个字段:

```typescript
@Post()
@ApiOperation({ summary: "创建患者" })
@ApiResponse({ type: CreatePatientResponseDto })
public async createPatient(
    @Body() createPatientRequest: CreatePatientRequestDto,
): Promise<CreatePatientResponseDto> {
    return this.patientsService.createPatient(createPatientRequest);
}
```

```typescript
/** 创建患者 */
  public async createPatient(patient: PatientEntity): Promise<Patient> {
    return this.patientRepository.create(patient);
  }
```

```
Argument of type 'CreatePatientRequestDto' is not assignable to parameter of type 'PatientEntity'.  
Type 'CreatePatientRequestDto' is missing the following properties from type 'PatientEntity': id, currentResearchStatusID, isActive, createdAt, updatedAt
```

1. 最简单的方案是使用 Partial 类型，修改 service 层的方法签名：

```typescript:apps/gcpm-backend/src/modules/patients/patients.controller.ts
// ... existing code ...
  public async createPatient(
    @Body() createPatientRequest: CreatePatientRequestDto,
  ): Promise<CreatePatientResponseDto> {
    // 移除手动设置的字段，让 service 层处理这些字段
    return this.patientsService.createPatient(createPatientRequest);
  }
// ... existing code ...
```

然后在 service 层：

```typescript:apps/gcpm-backend/src/modules/patients/patients.service.ts
// 使用 Partial<PatientEntity> 作为参数类型
public async createPatient(patientData: Partial<PatientEntity>): Promise<CreatePatientResponseDto> {
  const patient = {
    ...patientData,
    id: undefined,  // 这些字段在 service 层处理
    currentResearchStatusID: undefined,
    isActive: true,
    createdAt: new Date(),
    updatedAt: new Date(),
  };
  // ... 后续处理逻辑
}
```

1. 更规范的方案是使用专门的 mapper 类来处理 DTO 和实体之间的转换：

```typescript:apps/gcpm-backend/src/modules/patients/patients.mapper.ts
export class PatientMapper {
  static toEntity(dto: CreatePatientRequestDto): Partial<PatientEntity> {
    return {
      ...dto,
      id: undefined,
      currentResearchStatusID: undefined,
      isActive: true,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
  }
}
```

然后在 controller 中使用：

```typescript:apps/gcpm-backend/src/modules/patients/patients.controller.ts
// ... existing code ...
  public async createPatient(
    @Body() createPatientRequest: CreatePatientRequestDto,
  ): Promise<CreatePatientResponseDto> {
    const patientEntity = PatientMapper.toEntity(createPatientRequest);
    return this.patientsService.createPatient(patientEntity);
  }
// ... existing code ...
```

我个人更推荐第二种方案，因为：

1. 职责更清晰，数据转换逻辑集中在 mapper 类中
2. 更容易维护和测试
3. 如果将来需要添加更复杂的转换逻辑，可以直接在 mapper 中处理
4. 避免在多个地方重复相同的转换逻辑

这样的设计也更符合单一职责原则（SRP）和关注点分离原则。Controller 只负责接收请求和返回响应，Mapper 负责数据转换，Service 负责业务逻辑处理。
