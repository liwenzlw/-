你去超市买东西，结账的时候，收银员会将产品的条形码对着扫描口，一扫，那排数字就会显示在电脑里面，并连带出产品的图片、价格、库存量。

如下图是ESPRIT的一件衣服的吊牌，扫一下红框中的条形码，那排13位数字6902242429264就会出现在收银软件中，并带出商品图片、商品属性、库存量。而这13位数字6902242429264仅仅代表着款号为SC1203 颜色474尺码165（L）的这件货品（单品），如果是160（M）或者155（S），那么这13位数字又不一样了，后面的3位数字会发生变化，也就是理论上说的“库存的最小单位”，库存盘点就是对绑定这个条形码的商品库存量进行的；



**从上面的分析可以看出，SKU就是单款单色单码!**

![1564104376503](D:\myGitHubProject\notebook\项目实战\电商上面架构\SKU.assets\1564104376503.png)



**SKU就是库存的最小单位，正常情况是“单款单色单码”，国内品牌有把“单款单色”当做一个SKU、也有把“单款”的几个色当一个SKU、也有把一块面料的几个个款式当一个SKU,这些都是误读，其实你可以不用装得很SKU，就叫几个款色就行了；SKU是根据实际的库存盘点方案进行约定的**