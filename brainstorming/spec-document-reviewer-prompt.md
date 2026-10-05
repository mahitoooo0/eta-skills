# 规格文档评审者提示模板

在派发规格文档评审子代理时使用本模板。

**目的：** 核验规格完整、一致，可以进入实现规划。

**派发时机：** 规格文档已写入 `docs/superpowers/specs/` 之后。

在 Eta 里，这类评审通过 `delegate_task(worker=..., role="review", project=/workspace/<git 仓库根>, context=...)` 派发。评审子代理只能 `read_file` / `list_directory`，不能执行 shell、GUI 或浏览器；因此待评审的规格文件必须已经在该仓库（或其可读路径）中，并在 context 里给出其路径。评审子代理不得再派发子代理。

```
delegate_task(worker=<n>, role="review", project="/workspace/<git 仓库根>"):
  context: |
    你是规格文档评审者。核验这份规格是否完整、可以进入规划。

    **待评审规格：** [SPEC_FILE_PATH]

    ## 要检查什么

    | 类别 | 关注点 |
    |----------|------------------|
    | 完整性 | TODO、占位符、"TBD"、未完成的段落 |
    | 一致性 | 内部矛盾、相互冲突的要求 |
    | 清晰度 | 要求含糊到足以让人建错东西 |
    | 范围 | 是否聚焦到能装进单一计划——而不是覆盖多个相互独立的子系统 |
    | YAGNI | 未被要求的功能、过度设计 |

    ## 校准

    **只标出会在实现规划阶段造成真实问题的事项。**
    缺一节、一处矛盾、或一条含糊到能被两种方式理解的要求——这些是问题。措辞上的小改进、风格偏好、以及"有些节不如其他节详细"——这些不是。

    除非存在会导致计划有缺陷的严重缺口，否则通过。

    ## 输出格式

    ## Spec Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Section X]: [具体问题] - [为何它对规划重要]

    **Recommendations (advisory, do not block approval):**
    - [改进建议]
```

**评审者返回：** 状态、问题（如有）、建议。主代理用 `get_task_result` 取回结论。
