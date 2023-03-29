

    MATCH (n)
    OPTIONAL MATCH (n)-[r]-()
    DELETE n,r

## graph管理



CALL gds.graph.list

CALL gds.graph.drop('graph') YIELD graphName;



CALL gds.graph.project(

  'graph',                    

  ['Person', 'Book'],                             

  {KNOWS: {orientation: 'UNDIRECTED'},READ:{orientation: 'UNDIRECTED'}}

)

YIELD

  graphName AS graph,

  relationshipProjection AS knowsProjection,

  nodeCount AS nodes,

  relationshipCount AS rels



## 表示学习

CALL gds.beta.node2vec.stream('graph', {embeddingDimension: 2})

YIELD nodeId, embedding

RETURN nodeId, embedding



CALL gds.beta.node2vec.stream('graph', {embeddingDimension: 2})

YIELD nodeId, embedding

RETURN gds.util.asNode(nodeId), embedding

## Link prediction



链接预测是一个常见的机器学习任务，应用于图: 训练一个模型学习，在一个图中的节点对之间，在哪里应该存在关系。更准确地说，机器学习模型的输入是节点对的例子。在训练期间，节点对被标记为相邻或不相邻。



## 创建pipeline

CALL gds.beta.pipeline.linkPrediction.create('pipe')



## 列出pipeline

CALL gds.beta.pipeline.list()
YIELD pipelineName, pipelineType



## 删除pipeline

CALL gds.beta.pipeline.drop('pipe')
YIELD pipelineName, pipelineType





## 添加节点属性

CALL gds.beta.pipeline.linkPrediction.addNodeProperty('pipe', 'fastRP', {

  mutateProperty: 'embedding',

  embeddingDimension: 256,

  randomSeed: 42

})





## 添加链接功能



CALL gds.beta.pipeline.linkPrediction.addFeature('pipe', 'hadamard', {  nodeProperties: ['embedding', 'numberOfPosts'] }) YIELD featureSteps



## 配置关系拆分

CALL gds.beta.pipeline.linkPrediction.configureSplit('pipe', {
  testFraction: 0.25,
  trainFraction: 0.6,
  validationFolds: 3
})
YIELD splitConfig



## 增加候选模型

CALL gds.beta.pipeline.linkPrediction.addLogisticRegression('pipe', {penalty: 0.0625}) YIELD parameterSpace

CALL gds.alpha.pipeline.linkPrediction.addRandomForest('pipe', {numberOfDecisionTrees: 5})
YIELD parameterSpace

CALL gds.beta.pipeline.linkPrediction.addLogisticRegression('pipe', {maxEpochs: 500})
YIELD parameterSpace
RETURN parameterSpace.RandomForest AS randomForestSpace, parameterSpace.LogisticRegression AS logisticRegressionSpace



## 训练模型

CALL gds.beta.pipeline.linkPrediction.train('graph', {
  pipeline: 'pipe',
  modelName: 'lp-pipeline-model',
  randomSeed: 42
}) YIELD modelInfo
RETURN
  modelInfo.bestParameters AS winningModel,
  modelInfo.metrics.AUCPR.train.avg AS avgTrainScore,
  modelInfo.metrics.AUCPR.outerTrain AS outerTrainScore,
  modelInfo.metrics.AUCPR.test AS testScore



## 预测模型

CALL gds.beta.pipeline.linkPrediction.predict.stream('graph', {

  modelName: 'lp-pipeline-model',

  topN: 5,

  threshold: 0.45

})

 YIELD node1, node2, probability

 RETURN gds.util.asNode(node1).name AS person1, gds.util.asNode(node2).name AS person2, probability

 ORDER BY probability DESC, person1





> [Salute_Knowledge_Graph](https://github.com/Nicolas-gaofeng/Salute_Knowledge_Graph) ©2022 Powered By **GaoFeng Zhou**.
