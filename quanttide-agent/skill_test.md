啊，我懂了。你不是在问”怎么验证假设“，你是在问一个更根本的事：

我写了一个程序（那个聚类/抽象的代码），但我不知道它靠不靠谱，因为我连”正确答案“长什么样都没有感觉。

这是最难受的状态——你没法像写业务代码那样跑个单测，因为”Skill 应该怎么抽象“本身就没有 ground truth。


——

核心问题：没有标注数据，怎么知道程序没在胡说

你现在的处境其实是：

输入：一堆 Skill 文本
你的程序：输出一个层次化聚类 / 抽象结果
你：？？？这结果对不对？我怎么判断？


这不是你的问题，这是无监督方法 + 主观知识结构的固有困境。但有办法破。


——

破局思路：用你自己当”黄金标准“

你没有标注数据，但你有判断力。你读了那些 Skill，你知道哪些该合并、哪些不该。你缺的不是标准，是把你的标准外化出来的方法。

方法一：抽样人工对齐（最快建立感觉）

不要试图一次性评估全部结果。做这个：

1. 从聚类结果里随机抽 30 对 Skill

2. 对每一对，问自己：”这俩该合并吗？“ 只回答”该 / 不该 / 不确定“

3. 看你的程序和你的判断一致率

# 伪代码
pairs = random_pairs(skills, n=30)

for s_i, s_j in pairs:
    program_says = cluster_labels[i] == cluster_labels[j]  # 程序说该合并
    you_say = input(f”{s_i.name} vs {s_j.name}: 该合并吗？“)  # 你说
    
    results.append({
        ’pair‘: (s_i.name, s_j.name),
        ’program‘: program_says,
        ’human‘: you_say,
        ’agree‘: program_says == you_say
    })

print(f”一致率: {sum(r[’agree‘] for r in results) / len(results)}“)


一致率 > 70% → 程序基本靠谱，可以信任它的批量结果

一致率 50-70% → 方向对但阈值/权重要调

一致率 < 50% →  embedding 空间或相似度定义有问题

关键是：不一致的那些 pair，你逐条看为什么不一致。这比任何指标都管用——你会立刻发现程序在哪些地方”理解错了“。


——

方法二：稳定性测试（程序自己打自己）

无监督方法最怕的一件事：换个参数就完全不一样。

# 用不同参数跑 5 次
configs = [
    {’alpha‘: 0.5, ’beta‘: 0.3, ’gamma‘: 0.2, ’threshold‘: 0.6},
    {’alpha‘: 0.7, ’beta‘: 0.2, ’gamma‘: 0.1, ’threshold‘: 0.5},
    {’alpha‘: 0.3, ’beta‘: 0.5, ’gamma‘: 0.2, ’threshold‘: 0.7},
    # ...
]

all_results = []
for cfg in configs:
    labels = run_clustering(skills, **cfg)
    all_results.append(labels)

# 算不同参数下聚类结果的稳定性
from sklearn.metrics import adjusted_rand_score

stability_scores = []
for i in range(len(all_results)):
    for j in range(i+1, len(all_results)):
        ari = adjusted_rand_score(all_results[i], all_results[j])
        stability_scores.append(ari)

print(f”平均稳定性 ARI: {np.mean(stability_scores)}“)


• ARI > 0.8 → 程序很稳定，结果可信

• ARI 0.5-0.8 → 凑合，但某些边界 case 不稳定

• ARI < 0.5 → 结果基本是随机的，参数敏感

如果稳定性差，说明你的 embedding 空间本身信号就弱——这时候问题不在聚类算法，在你怎么表示 Skill。


——

方法三：消融测试（哪个部分在起作用）

你现在的程序里有好几层设计：意图 embedding、约束 embedding、动作 embedding、加权组合、层次聚类。哪个真的有用？

# 逐个关掉
variants = {
    ’full‘: lambda s: alpha*intent(s) + beta*constraint(s) + gamma*action(s),
    ’no_intent‘: lambda s: beta*constraint(s) + gamma*action(s),
    ’no_constraint‘: lambda s: alpha*intent(s) + gamma*action(s),
    ’no_action‘: lambda s: alpha*intent(s) + beta*constraint(s),
    ’text_only‘: lambda s: embed_raw_text(s),  # 纯文本 baseline
}

for name, vec_fn in variants.items():
    vecs = [vec_fn(s) for s in skills]
    labels = cluster(vecs)
    ari = adjusted_rand_score(labels, human_labels)  # 用你那30对标注
    print(f”{name}: 与人工判断一致率 = {ari}“)


这能告诉你：

• 如果 no_intent 掉得很厉害 → 意图层确实关键

• 如果 text_only 和 full 差不多 → 你花大力气做的结构化提取没用，白干了

• 如果 no_action 反而更好 → 动作细节在干扰抽象


——

方法四：反向生成测试（最狠的验证）

这是我最推荐的一个——让程序把抽象结果再”展开“，看能不能还原回原来的 Skill。

逻辑是：

如果抽象是对的，那么”抽象 Skill + 参数“应该能无损还原出原来的具体 Skill。

for cluster_id in set(labels):
    skills_in_cluster = [s for s, l in zip(all_skills, labels) if l == cluster_id]
    
    # 程序生成的抽象
    abstract = abstract_cluster(skills_in_cluster)
    
    # 用 LLM 尝试从抽象还原每个具体 Skill
    for s in skills_in_cluster:
        prompt = f”“”
        基于这个抽象 Skill：
        {abstract}
        
        加上这个变体的参数：
        {s[’constraints‘]}
        
        请还原出完整的操作步骤。
        “”“
        reconstructed = llm_generate(prompt)
        
        # 和原始 Skill 对比
        similarity = compare(reconstructed, s[’original_text‘])
        
    print(f”簇 {cluster_id}: 平均还原相似度 = {avg_sim}“)


• 还原相似度高（>0.8） → 抽象抓住了本质，信息没丢

• 还原相似度低 → 抽象要么过度压缩了，要么合并了不该合并的东西


——

你现在的”感觉“该怎么建立

说实话，你不需要一次性建立对”整个程序“的感觉。你需要的是分层建立信心：

层级	你要确认的事	方法
表示层	embedding 空间里相似的 Skill 在你看来确实相似吗？	随机抽 10 个最近邻，肉眼看
聚类层	同一个簇里的 Skill 你认同它们该在一起吗？	逐簇检查，标”合理/不合理“
抽象层	程序生成的抽象描述准确吗？	反向生成测试
合并层	合并后信息没丢吗？	还原测试

每一步你都能肉眼验证，每一步都在建立感觉。


——

一个实操建议

别从 800 行那个大 Skill 开始。先拿 10-15 个你最熟悉的 Skill（你手动写的 + 自动生成的各一半），人工标一遍”哪些该合并“。然后跑你的程序，看它和你的标注差多远。

这 15 个就是你的 development set。调参数、改 embedding、换聚类方法，都在这个集合上看效果。等你对它满意了，再放大到全量。


