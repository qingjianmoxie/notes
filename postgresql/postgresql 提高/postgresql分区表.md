postgresql支持分区表。老版本(即10版本以前)，使用的是继承方式实现分区表，需要定义父表、定义子表、定义子表约束、创建子表索引、创建分区插入删除修改函数和触发器等，操作复杂、需要注意的细节多。

## 继承实现分区表

步骤|说明
---|---
1|创建父表，如果父表上定义了约東，子表会继承，因此除非是全局约束，否则不应该在父表上定义约束，另外，父表不应该写入数据
2|通过 INHERITS方式创建继承表，也称之为子表或分区，子表的字段定义应该和父表保持一致。
3|给所有子表创建约束，只有满足约束条件的数据才能写入对应分区，注意分区约束值范围不要有重叠。
4|给所有子表创建索引，由于继承操作不会继承父表上的索引，因此索引需要手工创建。
5|在父表上定义 INSERT、 DELETE、 UPDATE触发器，将SQL分发到对应分区，这步可选，因为应用可以根据分区规则定位到对应分区进行DML操作。
6|启用 constraint exclusion参数，如果这个参数设置成of，则父表上的SQL性能会降低，后面会通过示例解释这个参数。

传统分区表的使用有以下注意事项：

1. 当往父表上插入数据时，需事先在父表上创建路由函数和触发器，数据才会根据分区键路由规则插入到对应分区中，目前仅支持范围分区和列表分区。
2. 分区表上的索引、约束需要使用单独的命令创建，目前没有办法一次性自动在所有分区上创建索引、约束。
3. 父表和子表允许单独定义主键，因此父表和子表可能存在重复的主键记录，目前不支持在分区表上定义全局主键。
4.  UPDATE时不建议更新分区键数据，特别是会使数据从一个分区移动到另一分区的场景，可通过更新触发器实现，但会带来管理上的成本。
5. 性能方面：传统分区表根据非分区键查询相比普通表性能差距较大，因为这种场景下分区表会扫描所有分区；根据分区键查询相比普通表性能有小幅降低，而查询分区表子表性能相比普通表略有提升

## 内置分区表
暂放