1. utils.py (核心逻辑实现)
- 添加了FSD所需的辅助函数:
  - compute_divergence: 这个函数用来计算两个概率分布之间的散度（例如JS散度、KL散度或余弦相似度），这是判断草稿模型和目标模型输出一致性的核心。
  - gather_draft_logits_for_candidates: 这个函数用于从草稿模型的输出(draft_logits)中，精确地提取出与投机生成的候选词元(candidates)相对应的logits。
- 修改了验证函数 evaluate_posterior:
  - 这是最关键的改动。我在这个函数里增加了一个新的逻辑分支，专门处理FSD。
  - 当启用了FSD（通过传入 fsd_div_threshold 参数），函数会利用上面添加的两个辅助函数计算草稿模型和目标模型logits的散度。
  - 然后，它会从第一个词元开始，逐个检查散度值，直到找到第一个超过预设阈值 (fsd_div_threshold) 的位置。在此之前的所有词元都会被接受。
  - 这样就实现了基于模型输出一致性的动态验收，而不是之前简单的逐个词元对比。
- 更新了函数签名以传递 draft_logits:
  - 为了让FSD逻辑能够拿到草稿模型的logits，我修改了 initialize_tree 和 update_inference_inputs 这两个函数，让它们在生成草稿树的同时，能够返回 draft_logits 这个张量。
2. cnets.py (草稿模型)
- 修改了 topK_genrate 函数:
  - 这个函数负责生成投机性草稿树。它在内部已经计算了每个草稿词元的logits。
  - 我修改了它的返回值，使其在返回草稿词元的同时，也一并返回这些词元对应的 draft_logits。这是整个数据流的起点。
3. ea_model.py (主生成循环)
- 为 ea_generate 方法添加了FSD参数:
  - 我在主生成函数 ea_generate 的入口添加了 fsd_div_threshold 和 fsd_div_type 这两个参数，这样用户在调用时就可以灵活启用和配置FSD。
- 打通了 draft_logits 的传递路径:
  - 在主生成循环中，我修改了对 initialize_tree、evaluate_posterior 和 update_inference_inputs 的调用。
  - 现在，从 initialize_tree 和 update_inference_inputs 获取到的 draft_logits 会被正确地捕获，并传递给 evaluate_posterior 函数用于FSD验证。
