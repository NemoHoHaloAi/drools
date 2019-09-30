# drools使用

服务端项目中使用drools的几个基本步骤；

## step 1 -- 添加相关依赖到maven pom.xml
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

## step 2 -- 创建实体类加载规则文件
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

## step 3 -- 创建DSLUtils类去执行相应规则
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

## step 4 -- 创建一个类去生成规则文件(比如生成 music.drl 的音乐规则文件，这一步是可选的，区别在于规则文件的生成是代码生成，还是人工生成，我们的项目中是运维同学在后台管理界面通过一些图形化输入框输入一些指定参数，而生成规则文件是服务端代码生成的，因此有了这部分，比较实用，一方面可以降低生成规则文件的门槛，任何人都可以做，另一方面也避免了人工出错的可能)
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

## step 4.1 -- 通过字符串创建文件，给上一步用的函数
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

## step 5 -- 规则文件加载，并用以检查当前用户是否满足下发规则条件
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
