---
title: 如何评估 AI 模型的效果
date: 2024-08-14 20:50:31
updated:
keywords:
slug: modelEvaluate
cover: /image/evaluate.png
top_image:
comments: false
maincolor:
categories:
  - ai 技术
tags:
  - 大模型
typora-root-url: ./modelEvaluate
---

# 1、概述

在工作中经常与 ai 模型算法同事打交道，有时他们推给我一个新模型或者优化的模型，我需要考虑如何评估它的效果，有没有达到我的业务要求。

借此机会，学习了一些评估 ai 模型效果的方式。

为了方便，下面提到的模型，如果没有特殊说明，都指的是 ai 模型。

模型有许多种分类，如大语言模型、检测模型、分类模型、预测模型等等，我们今天讨论的是检测模型给出的结果如何去评估。

# 2、模型输出的结果

> 用于评估分类模型性能的三个重要指标

## 2.1 准确率

指的是模型正确预测的样本数量与样本总体数量的比值。

> 准确率= 预测正确的样本数/样本总数

准确率关注的是总体的准确度，**关注所有样本集中的误检情况**

举例：有一批语音文件，我希望用模型检测出里面是否存在涉黄的语音，同时还希望正常的语音不能误判成涉黄，否则导致我产生人工复检的支出。
这个场景既关注涉黄语音的误检情况，也关注正常语音的误检情况，那么此时应该关注模型的准确率。

在100个语音文件中，当模型识别出 95个 文件正常， 5个文件存在涉黄，其中实际有3个是真的涉黄，剩下2个就是误检。
那么，准确率应该是 3+95 / 100 = 98%

在实际场景中，肯定更希望能找出所有的涉黄文件，那么就要使用召回率了

## 2.2 召回率

正例：我们所关注的预测结果，比如 车牌的号码识别，那么非车牌的输入图就是负例

指的是模型预测的所有正样例中，预测正确的样例与正样例的比值

> 召回率 = 预测正确的正例数量/所有输入的正例样本总数

**召回率强调模型的漏检情况**
如果模型能正确找出所有输入样本中的正例，那么意味着模型能100%识别出我们所关注的预测集（正例）

举例：
如果有一批影片截图，我希望用模型识别出所有存在法律风险的图。
如果模型能 100%识别出所有的存在风险的图，哪怕有些误检，它的召回率也是100%，就达到了我们的目的。

## 2.3 精确率

指的是模型预测的所有正例中，预测正确的正例，占所有预测出正例的集合的比值

> 精确率 = 预测正确的正例数量/预测结果中正例的总数

**精确率关注的是正例的误检情况**
假设有一批邮件，我希望能过滤出其中的垃圾邮件，允许一些误判，但是不能过滤掉正常的邮件。

这时正例是正常的邮件，关注正例的精确率，越高则代表越少的正常邮件被当作垃圾邮件过滤。

## 2.3 精确率召回率衡量的点

精确率和召回率的分子都是正例的识别正确数量，重点在分母。

召回率的分母是所有样本的正例，强调正例漏检的情况

精确率的分母是所有识别为正例的样本，其中可能包含负例被错识别为正例。这里强调的就是正例的准确度了。

正常来说，一般只关注模型的精确率和召回率，并通过调整置信度来控制检测阈值，扩大或调整判定的范围。
