

/data4/linming/.conda/envs/dnabert/lib/python3.6/site-packages/sklearn/metrics/_classification.py:1248: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.

## 二、同学意见

张俊寅：
MCC为0，可能是模型输出全1，或者标签对应错误。
宋程程：
max_length参数设置、建议试一下取固定上下文序列的长度。