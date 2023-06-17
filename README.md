#代码1：
package sydemo;

import jdk.nashorn.internal.scripts.JO;

import javax.swing.*;
import javax.swing.border.TitledBorder;
import javax.swing.text.BadLocationException;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.awt.event.WindowAdapter;
import java.awt.event.WindowEvent;
import java.awt.print.PrinterException;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.BindException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.ArrayList;
import java.util.StringTokenizer;

public class Server {
    private JFrame frame;  //创建Swing界面
    private JTextArea contentArea;  //文本域
    private JTextField txt_message;  //消息文本框
    private JTextField txt_max;     //人数上限文本框
    private JTextField txt_port;    //端口文本框
    private JButton btn_start;      //开始按钮
    private JButton btn_stop;       //停止按钮
    private JButton btn_send;       //发送按钮
    private JButton btn_out;       //导出按钮
    private JButton btn_change;   //切换主题按钮
    private JPanel northPanel;    //上方面板
    private JPanel southPanel;    //下方面板
    private JScrollPane rightPanel; //滚动面板--右
    private JScrollPane leftPanel;  //滚动面板--左
    private JSplitPane centerSplit; //分割面板（用户区和信息区）
    private JList<String> userList;  //列表面板
    private DefaultListModel<String> listModel;  //列表集合
    private boolean isStart;        //服务器是否启动的标志值
    private ServerSocket serverSocket;   //服务器socket
    private ServerThread serverThread;   //服务端发我线程
    private ArrayList<Server.ClientThread> clients;  //客户端连接线程
    private int themeIndex = 0; // 存储当前切换的主题编号
    private String[] themes = {"javax.swing.plaf.nimbus.NimbusLookAndFeel",
            "com.sun.java.swing.plaf.windows.WindowsClassicLookAndFeel",
            "javax.swing.plaf.metal.MetalLookAndFeel",
            "com.sun.java.swing.plaf.motif.MotifLookAndFeel"};    // 存储所有可供切换的LookAndFeel
    public Server(){     //构造方法
        frame = new JFrame("服务器");  //窗体名称
        contentArea = new JTextArea();     //实例化主聊天窗口
        contentArea.setEditable(false);    //设置该窗口不能被编辑
        contentArea.setForeground(Color.orange);    //设置字体颜色
        Font font = new Font("行楷",Font.PLAIN,20);   //设置字体和字体大小
        contentArea.setFont(font);     //应用
        txt_message = new JTextField();    //实例化输入框
        txt_max = new JTextField("30");    //设置人数上限文本框的显示值为30
        txt_port = new JTextField("666");  //实例化显示端口文本框，不要8808，port《=65535
        btn_start = new JButton("启动");  //实例化四个按钮
        btn_stop = new JButton("停止");
        btn_send = new JButton("发送");
        btn_out = new JButton("导出");
        btn_change = new JButton("切换主题");
        btn_stop.setEnabled(false);        //停止按钮默认状态为false（不可用）
        listModel = new DefaultListModel<String>();  //设置列表数据模型
        userList = new JList<String>(listModel);     //实例化列表窗体（存放数据模型）
        southPanel = new JPanel(new BorderLayout());  //实例化下方面板，布局为BorderLayout
        southPanel.setBorder(new TitledBorder("写消息"));  //设置该面板的名称
        southPanel.add(txt_message, "Center");  //添加信息输入框，居中
        southPanel.add(btn_send, "East");    //添加发送按钮，居右
        southPanel.add(btn_change,"West");   //添加切换按钮，居左
        leftPanel = new JScrollPane(userList);
        leftPanel.setBorder(new TitledBorder("在线用户"));
        rightPanel = new JScrollPane(contentArea);
        rightPanel.setBorder(new TitledBorder("消息显示区"));
        centerSplit = new JSplitPane(JSplitPane.HORIZONTAL_SPLIT, leftPanel, rightPanel);
        centerSplit.setDividerLocation(100);
        northPanel = new JPanel();
        northPanel.setLayout(new GridLayout(1, 7));
        northPanel.add(new JLabel("人数上限"));
        northPanel.add(txt_max);
        northPanel.add(new JLabel("端口"));
        northPanel.add(txt_port);
        northPanel.add(btn_start);
        northPanel.add(btn_stop);
        northPanel.add(btn_out);
        northPanel.setBorder(new TitledBorder("配置信息"));
        frame.setLayout(new BorderLayout());
        frame.add(northPanel, "North");
        frame.add(centerSplit, "Center");
        frame.add(southPanel, "South");
        frame.setSize(800, 600);
        int screen_w = Toolkit.getDefaultToolkit().getScreenSize().width;
        int screen_h = Toolkit.getDefaultToolkit().getScreenSize().height;
        frame.setLocation((screen_w-frame.getWidth())/6, (screen_h- frame.getHeight())/2);
        frame.setVisible(true);

        //设置主题
        btn_change.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                changeLookAndFeel();
            }
        });
        //关闭窗口时事件
        frame.addWindowListener(new WindowAdapter() {
            public void windowClosing(WindowEvent e) {
                if (isStart) {
                    closeServer();// 关闭服务器
                }
                System.exit(0);// 退出程序
            }
        });


        btn_start.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                if(isStart){
                    JOptionPane.showMessageDialog(frame, "服务器已启动，不要重复启动！",
                            "错误",JOptionPane.ERROR_MESSAGE);
                    return;
                }
                int max, port;
                try{
                    try{
                        max = Integer.parseInt(txt_max.getText());
                    }catch(Exception e1){
                        throw new Exception("人数上限必须为数字！");
                    }
                    if(max <= 0){
                        throw new Exception("人数上限为正整数！");
                    }
                    try{
                        port = Integer.parseInt(txt_port.getText());
                    }catch(Exception e1){
                        throw new Exception("端口号为正整数！");
                    }
                    if(port <= 0){
                        throw new Exception("端口号为正整数！");
                    }
                    serverStart(max, port);

                    contentArea.append("服务器已成功启动！人数上限："+max+"端口："+port+"\r\n");

                    JOptionPane.showMessageDialog(frame, "服务器成功启动！");

                    btn_start.setEnabled(false);
                    txt_max.setEnabled(false);
                    txt_port.setEnabled(false);

                    btn_stop.setEnabled(true);
                }catch (Exception ee){
                    JOptionPane.showMessageDialog(frame, ee.getMessage(), "错误", JOptionPane.ERROR_MESSAGE);
                }
            }
        });

        btn_stop.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                if(!isStart){
                    JOptionPane.showMessageDialog(frame, "服务器还未启动，无需停止！",
                            "错误",JOptionPane.ERROR_MESSAGE);
                    return;
                }
                try{
                    closeServer();

                    btn_start.setEnabled(true);
                    txt_max.setEnabled(true);
                    txt_port.setEnabled(true);

                    btn_stop.setEnabled(false);

                    contentArea.append("服务器成功停止！\r\n");

                    JOptionPane.showMessageDialog(frame, "服务器成功停止！");
                }catch (Exception exc){
                    JOptionPane.showMessageDialog(frame, "停止服务器异常！","错误",JOptionPane.ERROR_MESSAGE);
                }
            }
        });

        txt_message.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                try{
                    send();
                }catch (BadLocationException e1){
                    e1.printStackTrace();
                }
            }
        });

        btn_send.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent arg0) {
                try{
                    send();
                }catch (BadLocationException e){
                    e.printStackTrace();
                }
            }
        });

        btn_out.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                try{
                    outPort();
                }catch (PrinterException e1){
                    e1.printStackTrace();
                }
            }
        });
    }

    public void changeLookAndFeel() {
        String theme = themes[themeIndex];
        try {
            UIManager.setLookAndFeel(theme);
            SwingUtilities.updateComponentTreeUI(frame); //使用新LookAndFeel更新已有的Swing界面
        } catch (Exception e) {
            e.printStackTrace();
        }
        themeIndex++; // 切换主题编号自增
        if (themeIndex >= themes.length) {
            themeIndex = 0; // 超出主题数量时跳回第一个主题
        }
    }

    public void outPort() throws PrinterException{
        if(!isStart){
            JOptionPane.showMessageDialog(frame, "服务器未启动！");
        }else {
            contentArea.print();
        }
    }

    public void sendServerMessage(String message){
        for(int i = clients.size()-1;i >= 0;i--){
            clients.get(i).getWriter().println("服务器："+message+"(多人发送)");
            clients.get(i).getWriter().flush();
        }
    }
    public void send() throws BadLocationException{
        if(!isStart){
            JOptionPane.showMessageDialog(frame, "服务器还未启动，不能发送消息！",
                    "错误",JOptionPane.ERROR_MESSAGE);
            return;
        }

        if(clients.size() == 0){
            JOptionPane.showMessageDialog(frame, "没有用户在线，不能发送消息！",
                    "错误",JOptionPane.ERROR_MESSAGE);
            return;
        }

        String message = txt_message.getText().trim();

        if(message == null || message.equals("")){
            JOptionPane.showMessageDialog(frame, "消息不能为空！",
                    "错误",JOptionPane.ERROR_MESSAGE);
            return;
        }

        sendServerMessage(message);

        contentArea.append("服务器说："+txt_message.getText()+"\r\n");
        txt_message.setText(null);
    }
    public void serverStart(int max, int port) throws java.net.BindException{
        try{
            clients = new ArrayList<Server.ClientThread>();
            serverSocket = new ServerSocket(port);
            serverThread = new ServerThread(serverSocket, max);
            serverThread.start();
            isStart = true;
        }catch (BindException e){
            isStart = false;
            throw new BindException("端口号已被占用，请换一个！");
        }catch (Exception e1){
            e1.printStackTrace();
            isStart = false;
            throw new BindException("启动服务器异常！");
        }
    }
    public void closeServer(){
        try {
            if(serverThread != null)
                serverThread.stop();
            for (int i = clients.size() - 1; i >= 0; i--) {
                // 给所有在线用户发送关闭命令
                clients.get(i).getWriter().println("CLOSE");
                clients.get(i).getWriter().flush();
                // 释放资源
                clients.get(i).stop();// 停止此条为客户端服务的线程
                clients.get(i).reader.close();
                clients.get(i).writer.close();
                clients.get(i).socket.close();
                clients.remove(i);
            }
            if(serverSocket != null){
                serverSocket.close();
            }
            listModel.removeAllElements();
            isStart = false;
        }catch(IOException e){
            e.printStackTrace();
            isStart = true;
        }
    }
    class ServerThread extends Thread {
        private ServerSocket serverSocket;
        private int max;

        public ServerThread(ServerSocket serverSocket, int max) {
            this.serverSocket = serverSocket;
            this.max = max;
        }

        public void run() {
            //链接客服端
            while (true) {
                try {
                    Socket socket = serverSocket.accept();
                    System.out.println(clients.size());

                    if(clients.size() == max){

                        BufferedReader r = new BufferedReader(new
                                InputStreamReader(socket.getInputStream()));

                        PrintWriter w = new PrintWriter(socket.getOutputStream());

                        String inf = r.readLine();

                        StringTokenizer st = new StringTokenizer(inf, "@");

                        User user = new User(st.nextToken(), st.nextToken());

                        w.println("MAx@服务器：对不起，"+user.getName()+user.getIp()+"，服务器在线人数已达上限，请稍后尝试连接！");
                        w.flush();

                        r.close();
                        w.close();
                        socket.close();
                        continue;
                    }
                    ClientThread client = new ClientThread(socket);
                    client.start();
                    clients.add(client);

                    listModel.addElement(client.getUser().getName());

                    contentArea.append(client.getUser().getName()+client.getUser().getIp()+"上线！\r\n");
                }catch (IOException e){
                    e.printStackTrace();
                }
            }
        }
    }
    class ClientThread extends Thread {
        private Socket socket;
        private BufferedReader reader;
        private PrintWriter writer;
        private User user;
        public BufferedReader getReader() {return reader;}
        public PrintWriter getWriter() {return writer;}
        public User getUser() {return user;}

        public ClientThread(Socket socket) {
            try {
                this.socket = socket;

                reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

                writer = new PrintWriter(socket.getOutputStream());

                String inf = reader.readLine();

                StringTokenizer st = new StringTokenizer(inf, "@");

                user = new User(st.nextToken(), st.nextToken());

                writer.println(user.getName() + user.getIp() + "与服务器连接成功！");
                writer.println("使用[#用户名 聊天内容]可以进行私聊！");

                writer.flush();

                if (clients.size() > 0) {
                    String temp = "";

                    for (int i = clients.size() - 1; i >= 0; i--) {
                        temp += (clients.get(i).getUser().getName() + "/" +
                                clients.get(i).getUser().getIp()) + "@";
                    }
                    writer.println("USERLIST@" + clients.size() + "@" + temp);

                    writer.flush();
                }

                for (int i = clients.size() - 1; i >= 0; i--) {
                    clients.get(i).getWriter().println("ADD@" + user.getName() + user.getIp());

                    clients.get(i).getWriter().flush();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        public void run() {
            //连接客服端
            String message = null;
            while (true) {
                try {
                    message = reader.readLine();

                    if (message.equals("CLOSE")) {
                        contentArea.append(this.getUser().getName() + this.getUser().getIp() + "下线\r\n");

                        reader.close();
                        writer.close();
                        socket.close();

                        for (int i = clients.size() - 1; i >= 0; i--) {
                            clients.get(i).getWriter().println("DELETE@" + user.getName());

                            clients.get(i).getWriter().flush();
                        }

                        listModel.removeElement(user.getName());
                        for (int i = clients.size() - 1; i >= 0; i--) {
                            if (clients.get(i).getUser() == user) {
                                ClientThread temp = clients.get(i);
                                clients.remove(i);
                                temp.stop();
                                return;
                            }
                        }
                    } else {
                        dispatcherMessage(message);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        public void dispatcherMessage(String message) {
            StringTokenizer stringTokenizer = new StringTokenizer(message,"@");
            String source = stringTokenizer.nextToken();
            String owner = stringTokenizer.nextToken();
            String content = stringTokenizer.nextToken();

            message = source+"说："+content;
            contentArea.append(message+"\r\n");
            if(owner.equals("ALL")){
                for(int i = clients.size()-1;i>=0;i--){
                    clients.get(i).getWriter().println(message+"(对多人发送)");
                    clients.get(i).getWriter().flush();
                }
            }
        }
    }
}
  
  
#代码2：
  package sydemo;

import jdk.nashorn.internal.scripts.JO;
import sun.swing.text.html.FrameEditorPaneTag;

import javax.swing.*;
import javax.swing.border.TitledBorder;
import javax.swing.text.BadLocationException;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;
import java.util.StringTokenizer;

public class Client {
    private JFrame frame;
    private JList<String> userList;
    private JTextArea textArea;
    private JTextField textField;
    private JTextField txt_port;
    private JTextField txt_hostIp;
    private JTextField txt_name;
    private JButton btn_start;
    private JButton btn_stop;
    private JButton btn_send;
    private JButton btn_change;   //切换主题按钮
    private JPanel northPanel;
    private JPanel southPanel;
    private JScrollPane rightScroll;
    private JScrollPane leftScroll;
    private JSplitPane centerSplit;
    private DefaultListModel<String> listModel;
    private boolean isConnected = false;
    private Socket socket;
    private PrintWriter writer;
    private BufferedReader reader;
    private MessageThread messageThread;

    private Map<String, User> onLineUsers = new HashMap<String, User>();
    private int themeIndex = 0; // 存储当前切换的主题编号
    private String[] themes = {"javax.swing.plaf.nimbus.NimbusLookAndFeel",
            "com.sun.java.swing.plaf.windows.WindowsClassicLookAndFeel",
            "javax.swing.plaf.metal.MetalLookAndFeel",
            "com.sun.java.swing.plaf.motif.MotifLookAndFeel"};    // 存储所有可供切换的LookAndFeel

    public Client(){
        textArea = new JTextArea();
        textArea.setEditable(false);
        textArea.setForeground(Color.GREEN);
        Font font = new Font("行楷",Font.PLAIN,18);   //设置字体和字体大小
        textArea.setFont(font);     //应用
        textField = new JTextField();

        txt_port = new JTextField("666");
        txt_hostIp = new JTextField("127.0.0.1");
        txt_name = new JTextField("卜");

        btn_start = new JButton("连接");
        btn_stop = new JButton("断开");
        btn_send = new JButton("发送");
        btn_change = new JButton("切换主题");
        listModel = new DefaultListModel<String>();
        userList = new JList<String>(listModel);
        northPanel = new JPanel();
        northPanel.setLayout(new GridLayout(1, 7));
        northPanel.add(new JLabel("端口"));
        northPanel.add(txt_port);
        northPanel.add(new JLabel("服务器IP"));
        northPanel.add(txt_hostIp);
        northPanel.add(new JLabel("姓名"));
        northPanel.add(txt_name);
        northPanel.add(btn_start);
        northPanel.add(btn_stop);

        northPanel.setBorder(new TitledBorder("连接信息"));
        rightScroll = new JScrollPane(textArea);
        rightScroll.setBorder(new TitledBorder("消息显示区"));
        leftScroll = new JScrollPane(userList);
        leftScroll.setBorder(new TitledBorder("在线用户"));
        southPanel = new JPanel(new BorderLayout());
        southPanel.add(textField, "Center");
        southPanel.add(btn_send, "East");
        southPanel.add(btn_change,"West");   //添加切换按钮，居左
        southPanel.setBorder(new TitledBorder("写消息"));

        centerSplit = new JSplitPane(JSplitPane.HORIZONTAL_SPLIT, leftScroll, rightScroll);
        centerSplit.setDividerLocation(100);
        frame = new JFrame("客户机");
        frame.setLayout(new BorderLayout());

        frame.add(northPanel, "North");
        frame.add(centerSplit, "Center");
        frame.add(southPanel, "South");
        frame.setSize(800, 600);

        int screen_w = Toolkit.getDefaultToolkit().getScreenSize().width;
        int screen_h = Toolkit.getDefaultToolkit().getScreenSize().height;
        frame.setLocation((screen_w-frame.getWidth()*7/6), (screen_h- frame.getHeight())/2);
        frame.setVisible(true);

        //设置主题
        btn_change.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                changeLookAndFeel();
            }
        });

        btn_start.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                int port;

                if(isConnected){

                    JOptionPane.showMessageDialog(frame,"已处于连接上状态，不要重复连接！",
                            "错误",JOptionPane.ERROR_MESSAGE);
                    return;
                }
                try{
                    try{
                        port = Integer.parseInt(txt_port.getText().trim());
                    }catch (NumberFormatException e2){
                        throw new Exception("端口号不符合要求！端口号为整数！");
                    }
                    String hostIp = txt_hostIp.getText().trim();

                    String name = txt_name.getText().trim();

                    if(name.equals("") || hostIp.equals("")){
                        throw new Exception("姓名，服务器IP不能为空！");
                    }
                    boolean flag = connectServer(port, hostIp, name);
                    if(flag == false){
                        throw new Exception("与服务器连接失败！");
                    }

                    frame.setTitle(name);

                    JOptionPane.showMessageDialog(frame, "连接成功!");
                }catch (Exception exc){

                    JOptionPane.showMessageDialog(frame, exc.getMessage(),
                            "错误", JOptionPane.ERROR_MESSAGE);
                }
            }
        });

        btn_stop.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                if (!isConnected) {
                    JOptionPane.showMessageDialog(frame, "已处于断开状态，不要重复断开!",
                            "错误", JOptionPane.ERROR_MESSAGE);
                    return;
                }
                try {
                    boolean flag = closeConnection();// 断开连接
                    if (flag == false) {
                        throw new Exception("断开连接发生异常！");
                    }
                    JOptionPane.showMessageDialog(frame, "成功断开!");
                } catch (Exception exc) {
                    JOptionPane.showMessageDialog(frame, exc.getMessage(),
                            "错误", JOptionPane.ERROR_MESSAGE);
                }
            }
        });

        textField.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                send();
            }
        });

        btn_send.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent arg0) {
                send();
            }
        });
    }

    public void changeLookAndFeel() {
        String theme = themes[themeIndex];
        try {
            UIManager.setLookAndFeel(theme);
            SwingUtilities.updateComponentTreeUI(frame); //使用新LookAndFeel更新已有的Swing界面
        } catch (Exception e) {
            e.printStackTrace();
        }
        themeIndex++; // 切换主题编号自增
        if (themeIndex >= themes.length) {
            themeIndex = 0; // 超出主题数量时跳回第一个主题
        }
    }
    public void send(){
        if(!isConnected){
            JOptionPane.showMessageDialog(frame,"还未连接服务器，无法发送信息！",
                         "错误", JOptionPane.ERROR_MESSAGE);
            return;
        }
        String message = textField.getText().trim();
        if(message == null || message.equals("")){
            JOptionPane.showMessageDialog(frame, "消息不能为空！","错误",JOptionPane.ERROR_MESSAGE);
            return;
        }
        sendMessage(frame.getTitle()+"@"+"ALL"+"@"+message);
        textField.setText(null);
    }

    public synchronized boolean closeConnection() {
        try {
            sendMessage("CLOSE");// 发送断开连接命令给服务器
            messageThread.stop();// 停止接受消息线程
            // 释放资源
            if (reader != null) {
                reader.close();
            }
            if (writer != null) {
                writer.close();
            }
            if (socket != null) {
                socket.close();
            }
            isConnected = false;
            return true;
        } catch (IOException e1) {
            e1.printStackTrace();
            isConnected = true;
            return false;
        }
    }

    public boolean connectServer(int port, String hostIp, String name) throws
            BadLocationException{
        try{
            socket = new Socket(hostIp, port);
            writer = new PrintWriter(socket.getOutputStream());
            reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

            sendMessage(name+"@"+socket.getLocalAddress().toString());

            messageThread = new MessageThread(reader, textArea);
            messageThread.start();
            isConnected = true;
            return true;
        }catch (Exception e){
            textArea.append("与端口号为："+port+"  IP地址为："+hostIp+"  的服务器连接失败！"+"\r\n");
            isConnected = false;
            return false;
        }
    }
    public void sendMessage(String message){
        writer.println(message);
        writer.flush();
    }

    class MessageThread extends Thread{
        private BufferedReader reader;
        private JTextArea textArea;

        public MessageThread(BufferedReader reader, JTextArea textArea){
            this.reader = reader;
            this.textArea = textArea;
        }
        public synchronized void closeCon() throws Exception{
            listModel.removeAllElements();

            if(reader != null){
                reader.close();
            }
            if(writer != null){
                writer.close();
            }
            if(socket != null){
                socket.close();
            }
            isConnected = false;
        }

        public void run(){
            String message = "";
            while (true){
                try{
                    message = reader.readLine();
                    StringTokenizer stringTokenizer = new StringTokenizer(message, "/@");
                    String command = stringTokenizer.nextToken();
                    if(command.equals("CLOSE")){
                        textArea.append("服务器已关闭！\r\n");
                        closeCon();
                        return;
                    }else if(command.equals("ADD")){
                        String username = "";
                        String userIp = "";
                        if((username = stringTokenizer.nextToken()) != null &&
                                (userIp = stringTokenizer.nextToken()) != null){
                            User user = new User(username, userIp);
                            onLineUsers.put(username, user);
                            listModel.addElement(username);
                        }
                    }else if(command.equals("DELETE")){
                        String username = stringTokenizer.nextToken();
                        User user = (User) onLineUsers.get(username);
                        onLineUsers.remove(user);
                        listModel.removeElement(username);
                    }else if(command.equals("USERLIST")){
                        int size = Integer.parseInt((stringTokenizer.nextToken()));
                        String username = null;
                        String userIp = null;
                        for(int i = 0;i < size; i++){
                            username = stringTokenizer.nextToken();
                            userIp = stringTokenizer.nextToken();
                            User user = new User(username, userIp);
                            onLineUsers.put(username, user);
                            listModel.addElement(username);
                        }
                    }else if(command.equals("MAX")){
                        textArea.append(stringTokenizer.nextToken()+stringTokenizer.nextToken()+"\r\n");
                        closeCon();
                        JOptionPane.showMessageDialog(frame, "服务器缓冲区已满！",
                                "错误", JOptionPane.ERROR_MESSAGE);
                        return;
                    }else {
                        textArea.append(message+"\r\n");
                    }
                }catch (IOException e){
                    e.printStackTrace();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }
        public boolean connectServer(int port, String hostIp, String name) throws
                BadLocationException{
            try{
                socket = new Socket(hostIp, port);
                writer = new PrintWriter(socket.getOutputStream());
                reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

                sendMessage(name+"@"+socket.getLocalAddress().toString());

                messageThread = new MessageThread(reader, textArea);
                messageThread.start();
                isConnected = true;
                return true;
            }catch (Exception e){
                textArea.append("与端口号为："+port+" IP地址为："+hostIp+" 的服务器连接失败！"+"\r\n");
                isConnected = false;
                return false;
            }
        }

    }

}
