---
title: 将Throwable的信息转换为字符串
tags: 开发tips
---
``` java
  /**
	 * 将异常信息转化成字符串
	 * @param t
	 * @return
	 * @throws IOException 
	 */
	public static String throwable2Str(Throwable t) throws IOException{
	    if(t == null)
	        return "";
	    ByteArrayOutputStream os = new ByteArrayOutputStream();
	    try{
	        t.printStackTrace(new PrintStream(os));
	    }finally{
	        os.close();
	    }
	    return os.toString();
	}
  ```
