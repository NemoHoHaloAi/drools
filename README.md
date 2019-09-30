# drools规则引擎

## 关于

drools是一款标准、效率高、速度快的开源规则引擎，基于ReteOO算法，目前主要应用场景在广告、活动下发等领域非常多，比如APP的活动下发，通常都是有很多条件限制的，且各种活动层出不穷，无法代码穷举，而如果每次为了一个活动重新发版上线，显然是不合理的，因此通过drools将活动中变的部分抽象为一个个单独的规则文件，来屏蔽这部分的变化，使得系统不需要从代码层面做出改变，当然了为了更加极致的抽象，通常还需要对规则中的一些可配条件（大于、小于、等于、范围、次数等）也提取到数据库中，这样在现有规则不满足要求时，可以直接通过更改数据库的对应规则表来完善，同样不需要改代码；

我们当时的需求主要就是广告、活动下发规则比较多，广告也是各式各样，因此去调研了drools，对drools也没有过多的挖掘其更多特性，因此还需要大家的指点；

## drools简单使用

服务端项目中使用drools的几个基本步骤；

### step 1 -- 添加相关依赖到maven pom.xml
```
<dependency>
	<groupId>org.drools</groupId>
	<artifactId>drools-core</artifactId>
	<version>6.4.0.Final</version>
</dependency>
<dependency>
	<groupId>org.drools</groupId>
	<artifactId>drools-compiler</artifactId>
	<version>6.4.0.Final</version>
</dependency>
```

### step 2 -- 创建实体类加载规则文件
```
public class CarIllegalRules extends BaseRules{
	
	public static void main(String[] args) {
		try {
			KieServices ks = KieServices.Factory.get();  
            		KieContainer kContainer = ks.getKieClasspathContainer();  
        	    	KieSession ksession = kContainer.newKieSession("ksession-rules");
	        
	 	        CarIllegalRules carIllegalRules = new CarIllegalRules(10,500,10);
	 	        ksession.insert(carIllegalRules);
		        ksession.fireAllRules();
		        System.out.println(carIllegalRules.isCan_push()+"，"+carIllegalRules.getContent());    
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	private int illegal_count;
	
	private int illegal_money;
	
	private int illegal_points;

	public CarIllegalRules(int illegal_count, int illegal_money, int illegal_points) {
		super();
		this.illegal_count = illegal_count;
		this.illegal_money = illegal_money;
		this.illegal_points = illegal_points;
		this.param_value = "illegal_count,illegal_money,illegal_points";
	}

	@Override
	public String toString() {
		return "CarIllegalRules [illegal_count=" + illegal_count + ", illegal_money=" + illegal_money
				+ ", illegal_points=" + illegal_points + ", can_push=" + can_push + ", content=" + content + ", tts="
				+ tts + "]";
	}

	public int getIllegal_count() {
		return illegal_count;
	}

	public void setIllegal_count(int illegal_count) {
		this.illegal_count = illegal_count;
	}

	public int getIllegal_money() {
		return illegal_money;
	}

	public void setIllegal_money(int illegal_money) {
		this.illegal_money = illegal_money;
	}

	public int getIllegal_points() {
		return illegal_points;
	}

	public void setIllegal_points(int illegal_points) {
		this.illegal_points = illegal_points;
	}
}
```
> PS:main函数是用来测试这个类的；

### step 3 -- 创建DSLUtils类去执行相应规则
```
public class DSLUtil {
	public static void fireRules(File file, Object rules) {
		try {
			KieServices kieServices = KieServices.Factory.get();
			KieFileSystem kfs = kieServices.newKieFileSystem();
			Resource resource = kieServices.getResources().newFileSystemResource(file);
			fire(rules, kieServices, kfs, resource);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	public static void fireRules(String urlStr, Object rules) {
		try {
			KieServices kieServices = KieServices.Factory.get();
			KieFileSystem kfs = kieServices.newKieFileSystem();
			Resource resource = kieServices.getResources().newFileSystemResource(FileUtil.getFileFromUrl(urlStr));
			fire(rules, kieServices, kfs, resource);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	private static void fire(Object commonRules, KieServices kieServices, KieFileSystem kfs, Resource resource)
			throws Exception {
		resource.setResourceType(ResourceType.DRL);
		kfs.write(resource);
		KieBuilder kieBuilder = kieServices.newKieBuilder(kfs).buildAll();
		if (kieBuilder.getResults().getMessages(Message.Level.ERROR).size() > 0) {
			throw new Exception();
		}
		KieContainer kieContainer = kieServices.newKieContainer(kieServices.getRepository().getDefaultReleaseId());
		KieBase kBase = kieContainer.getKieBase();
		KieSession ksession = kBase.newKieSession();
		ksession.insert(commonRules);
		ksession.fireAllRules();
	}
}
```

### step 4 -- 创建一个类去生成规则文件

比如生成 music.drl 的音乐规则文件，这一步是可选的，区别在于规则文件的生成是代码生成，还是人工生成，我们的项目中是运维同学在后台管理界面通过一些图形化输入框输入一些指定参数，而生成规则文件是服务端代码生成的，因此有了这部分，比较实用，一方面可以降低生成规则文件的门槛，任何人都可以做，另一方面也避免了人工出错的可能；

```
public class ActivityUtil {
	/**
	 * rule template string
	 */
	private static String template = 
		"package com.aispeech.dsl\r\n\r\n" + 
		"import {entity_package_path};\r\n\r\n" +
		"import {entity_package_path}.*;\r\n\r\n" +
		"rule \"{rule_name}\"\r\n\r\n" +
		"when\r\n" +
		"\t{instance_name}:{class_name}({rules})\r\n" +
		"then\r\n" +
		"\t{do}\r\n" +
		"end";
	
	private static final String AND = " && ";
	private static final String OR = " || ";

	/**
	 * get business rule file xxx.drl
	 * @param carActivity user info entity
	 * @param clazz entity class
	 * @return
	 */
	public static File createBusinessRuleFile(Car_activity carActivity, Class clazz, String[] param_texts, String[] param_values) {
		String ruleStr = template;
		String entity_package_path = (clazz+"").substring(6);
		String rule_name = "rule_"+carActivity.getId();
		String class_name = (clazz+"").substring((clazz+"").lastIndexOf(".")+1);
		String instance_name = class_name.toLowerCase();
		
		String rules = "";
		JSONArray conditionArray = JSONArray.parseArray(carActivity.getAim_condition());
		for(int i=0;i<conditionArray.size();i++) {
			JSONObject condition = conditionArray.getJSONObject(i);
			rules += "\r\n\t\t("+condition.getString("param")+condition.getString("operator")+condition.getString("value")+")" + AND;
		}
		rules = rules.length()>0?rules.substring(0, rules.lastIndexOf(AND)):rules;
		
		for (String param_value : param_values) {
			rules += "\r\n\t\t,"+param_value.toLowerCase()+":"+param_value;
		}
		
		String content = JSONObject.parseObject(carActivity.getContent()).getString("content");
		String tts = carActivity.getTts();
		for (int i=0;i<param_texts.length;i++) {
			content = content.replace("#"+param_texts[i]+"#", "\"+"+param_values[i]+"+\"");
			tts = tts.replace("#"+param_texts[i]+"#",  "\"+"+param_values[i]+"+\"");
		}
		String _do = instance_name+".setCan_push(true);";
		_do += "\r\n\t" + instance_name+".setContent(\""+content+"\");";
		_do += "\r\n\t" + instance_name+".setTts(\""+tts+"\");";
		
		return returnFile(ruleStr, entity_package_path, rule_name, class_name, instance_name, _do, rules);
	}

	/**
	 * @param ruleStr
	 * @param entity_package_path
	 * @param rule_name
	 * @param class_name
	 * @param instance_name
	 * @param _do
	 * @param rules
	 * @return
	 */
	private static File returnFile(String ruleStr, String entity_package_path, String rule_name, String class_name,
			String instance_name, String _do, String rules) {
		ruleStr = ruleStr.replace("{entity_package_path}", entity_package_path)
				.replace("{rule_name}", rule_name)
				.replace("{class_name}", class_name)
				.replace("{instance_name}", instance_name)
				.replace("{do}", _do)
				.replace("{rules}", rules);
		System.out.println(ruleStr);
		return FileUtil.getFileFromText(rule_name, ".drl", ruleStr);
	}
}
```

### step 4.1 -- 通过字符串创建文件，给上一步用的函数
```
public static File getFileFromText(String tempFileName, String fileTail, String text) {
	try {
		File file = File.createTempFile(tempFileName, fileTail);
		FileOutputStream fos = new FileOutputStream(file);
		fos.write(text.getBytes());
		if(fos!=null){
		    fos.close();
		}
		return file;
	} catch (FileNotFoundException e) {
		e.printStackTrace();
	} catch (IOException e) {
		e.printStackTrace();
	}
	return null;
}
```

### step 5 -- 规则文件加载，并用以检查当前用户是否满足下发规则条件
```
BaseRules baseRules = new CarIllegalRules(count, money, points);
if(baseRules!=null) {
	logger.info("before fire rules:"+baseRules);
	DSLUtil.fireRules(ActivityUtil.createBusinessRuleFile(car_activity, baseRules.getClass(), 
		baseRules.getParam_text().split(","), baseRules.getParam_value().split(",")), baseRules);
	logger.info("after fire rules:"+baseRules);
	if(baseRules.isCan_push()) {
        	//In here, the rules are used to judge the success of the entity, and you can do something!!!
    	}
}
```

## 小结

本文通过对drools的简单使用步骤的讲解，为大家展示了drools最简单的使用方式，而它能做到的远远不止看到的这些，但是基本框架是这样，大家可以尝试挖掘规则文件的一些黑操作，可以对多变的业务进行极致的抽象，再也不用为了这些重新发版啦，LOL；

PS：想深入了解的同学还是要去看看Rete算法、drools的推理机制等等，本文主要从该引擎的入门出发哈；
