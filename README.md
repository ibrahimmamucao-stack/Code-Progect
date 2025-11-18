package staffmanagementsystem;

import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.*;
import java.time.LocalDate;
import java.util.HashMap;
import java.util.List;
import java.util.ArrayList;
import java.util.Map;

class LeaveRequest implements Serializable {
    private LocalDate startDate;
    private LocalDate endDate;
    private String reason;
    private String status;

    public LeaveRequest(LocalDate startDate, LocalDate endDate, String reason) {
        this.startDate = startDate;
        this.endDate = endDate;
        this.reason = reason;
        this.status = "Pending"; // Start as Pending
    }

    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate() { return endDate; }
    public String getReason() { return reason; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}

class Staff implements Serializable {
    private int id;
    private String name;
    private String position;
    private double salary;
    private Map<LocalDate, String> attendance;
    private List<LeaveRequest> leaveRequests;

    public Staff(int id, String name, String position, double salary) {
        this.id = id;
        this.name = name;
        this.position = position;
        this.salary = salary;
        this.attendance = new HashMap<>();
        this.leaveRequests = new ArrayList<>();
    }

    public int getId() { return id; }
    public String getName() { return name; }
    public String getPosition() { return position; }
    public double getSalary() { return salary; }

    public void setName(String name) { this.name = name; }
    public void setPosition(String position) { this.position = position; }
    public void setSalary(double salary) { this.salary = salary; }

    public Map<LocalDate, String> getAttendance() { return attendance; }
    public void markAttendance(LocalDate date, String status) {
        attendance.put(date, status);
    }

    public List<LeaveRequest> getLeaveRequests() { return leaveRequests; }
    public void addLeaveRequest(LeaveRequest request) {
        leaveRequests.add(request);
    }
}

class StaffManager {
    private List<Staff> staffList = new ArrayList<>();
    private int nextId = 1;
    private static final String DATA_FILE = "staffData.ser";

    public StaffManager() {
        loadData();
        if (staffList.isEmpty()) {
            nextId = 1; // Reset if no data
        } else {
            nextId = staffList.stream().mapToInt(Staff::getId).max().orElse(0) + 1;
        }
    }

    public void addStaff(String name, String position, double salary) {
        staffList.add(new Staff(nextId++, name, position, salary));
        saveData();
    }

    public List<Staff> getAllStaff() { return staffList; }

    public boolean updateStaff(int id, String name, String position, double salary) {
        Staff staff = findStaffById(id);
        if (staff == null) return false;
        staff.setName(name);
        staff.setPosition(position);
        staff.setSalary(salary);
        saveData();
        return true;
    }

    public boolean deleteStaff(int id) {
        boolean removed = staffList.removeIf(s -> s.getId() == id);
        if (removed) saveData();
        return removed;
    }

    public Staff findStaffById(int id) {
        return staffList.stream().filter(s -> s.getId() == id).findFirst().orElse(null);
    }

    public boolean markAttendance(int id, LocalDate date, String status) {
        Staff staff = findStaffById(id);
        if (staff == null) return false;
        staff.markAttendance(date, status);
        saveData();
        return true;
    }

    public boolean addLeaveRequest(int id, LocalDate start, LocalDate end, String reason) {
        Staff staff = findStaffById(id);
        if (staff == null) return false;
        staff.addLeaveRequest(new LeaveRequest(start, end, reason));
        saveData();
        return true;
    }

    public boolean approveLeaveRequest(int staffId, int requestIndex) {
        Staff staff = findStaffById(staffId);
        if (staff == null || requestIndex < 0 || requestIndex >= staff.getLeaveRequests().size()) return false;
        staff.getLeaveRequests().get(requestIndex).setStatus("Approved");
        saveData();
        return true;
    }

    private void saveData() {
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(DATA_FILE))) {
            oos.writeObject(staffList);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @SuppressWarnings("unchecked")
    private void loadData() {
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(DATA_FILE))) {
            staffList = (List<Staff>) ois.readObject();
        } catch (FileNotFoundException e) {
            // File doesn't exist yet, start with empty list
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

class LoginSystem extends JFrame {
    private JTextField usernameField;
    private JPasswordField passwordField;
    private JButton loginButton;
    private JLabel messageLabel;

    // Simulated user database (username -> {password, role})
    private static final Map<String, String[]> users = new HashMap<>();
    static {
        users.put("superadmin", new String[]{"superpass", "super admin"});
        users.put("admin", new String[]{"adminpass", "admin"});
        users.put("user", new String[]{"userpass", "user"});
    }

    public LoginSystem() {
        setTitle("Login System");
        setSize(300, 200);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new GridLayout(4, 1));

        usernameField = new JTextField();
        passwordField = new JPasswordField();
        loginButton = new JButton("Login");
        messageLabel = new JLabel("", SwingConstants.CENTER);

        add(new JLabel("Username:"));
        add(usernameField);
        add(new JLabel("Password:"));
        add(passwordField);
        add(loginButton);
        add(messageLabel);

        loginButton.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                String username = usernameField.getText().trim();
                String password = new String(passwordField.getPassword());

                if (username.isEmpty() || password.isEmpty()) {
                    messageLabel.setText("Please fill in all fields.");
                    return;
                }

                if (users.containsKey(username)) {
                    if (users.get(username)[0].equals(password)) {
                        String role = users.get(username)[1];
                        messageLabel.setText("Login successful!");
                        openStaffGUI(role);
                    } else {
                        messageLabel.setText("You inserted an invalid password.");
                    }
                } else {
                    messageLabel.setText("Invalid username.");
                }
            }
        });

        setVisible(true);
    }

    private void openStaffGUI(String role) {
        new StaffGUI(role).setVisible(true);
        this.dispose();
    }
}

class StaffGUI extends JFrame {
    private StaffManager manager;
    private JTable staffTable, attendanceTable, leaveTable;
    private DefaultTableModel staffModel, attendanceModel, leaveModel;
    private JTextField idField, nameField, positionField, salaryField;
    private JButton addBtn, updBtn, delBtn, searchBtn, markBtn, submitBtn, approveBtn;
    private String userRole;
    private int userId = -1; // For user role, restrict to their own ID

    public StaffGUI(String role) {
        this.userRole = role;
        manager = new StaffManager(); // Loads data on creation
        if (role.equals("user")) {
            userId = 1; // Simulate user ID (in real app, link to logged-in user)
        }
        setupUI();
    }

    private void setupUI() {
        setTitle("Staff Management System - " + userRole);
        setSize(900, 600);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        JTabbedPane tabs = new JTabbedPane();
        tabs.add("Staff", createStaffTab());
        tabs.add("Attendance", createAttendanceTab());
        tabs.add("Leave Requests", createLeaveTab());

        JButton logoutBtn = new JButton("Logout");
        logoutBtn.addActionListener(e -> {
            this.dispose();
            new LoginSystem();
        });
        add(logoutBtn, BorderLayout.SOUTH);

        add(tabs);
        refreshTables();
        applyRoleRestrictions();
    }

    private JPanel createStaffTab() {
        JPanel panel = new JPanel(new BorderLayout());

        staffModel = new DefaultTableModel(new String[]{"ID", "Name", "Position", "Salary"}, 0);
        staffTable = new JTable(staffModel);
        panel.add(new JScrollPane(staffTable), BorderLayout.CENTER);

        JPanel bottom = new JPanel(new FlowLayout());
        idField = new JTextField(5);
        nameField = new JTextField(10);
        positionField = new JTextField(10);
        salaryField = new JTextField(10);

        bottom.add(new JLabel("ID:")); bottom.add(idField);
        bottom.add(new JLabel("Name:")); bottom.add(nameField);
        bottom.add(new JLabel("Position:")); bottom.add(positionField);
        bottom.add(new JLabel("Salary:")); bottom.add(salaryField);

        addBtn = new JButton("Add");
        updBtn = new JButton("Update");
        delBtn = new JButton("Delete");
        searchBtn = new JButton("Search");

        addBtn.addActionListener(e -> addStaff());
        updBtn.addActionListener(e -> updateStaff());
        delBtn.addActionListener(e -> deleteStaff());
        searchBtn.addActionListener(e -> searchStaff());

        bottom.add(addBtn);
        bottom.add(updBtn);
        bottom.add(delBtn);
        bottom.add(searchBtn);

        panel.add(bottom, BorderLayout.SOUTH);
        return panel;
    }

    private JPanel createAttendanceTab() {
        JPanel panel = new JPanel(new BorderLayout());

        attendanceModel = new DefaultTableModel(new String[]{"Date", "Status"}, 0);
        attendanceTable = new JTable(attendanceModel);
        panel.add(new JScrollPane(attendanceTable), BorderLayout.CENTER);

        markBtn = new JButton("Mark Today");
        markBtn.addActionListener(e -> markAttendance());

        panel.add(markBtn, BorderLayout.SOUTH);
        return panel;
    }

    private JPanel createLeaveTab() {
        JPanel panel = new JPanel(new BorderLayout());

        leaveModel = new DefaultTableModel(new String[]{"Start Date", "End Date", "Reason", "Status"}, 0);
        leaveTable = new JTable(leaveModel);
        panel.add(new JScrollPane(leaveTable), BorderLayout.CENTER);

        JPanel bottom = new JPanel(new FlowLayout());
        submitBtn = new JButton("Submit Leave");
        approveBtn = new JButton("Approve Leave");

        submitBtn.addActionListener(e -> submitLeaveRequest());
        approveBtn.addActionListener(e -> approveLeaveRequest());

        bottom.add(submitBtn);
        bottom.add(approveBtn);
        panel.add(bottom, BorderLayout.SOUTH);

        return panel;
    }

    private void applyRoleRestrictions() {
        if (userRole.equals("user")) {
            // User: Only attendance and leave
            addBtn.setEnabled(false);
            updBtn.setEnabled(false);
            delBtn.setEnabled(false);
            searchBtn.setEnabled(false);
            approveBtn.setEnabled(false);
            idField.setText(String.valueOf(userId));
            idField.setEditable(false);
            nameField.setEditable(false);
            positionField.setEditable(false);
            salaryField.setEditable(false);
        } else if (userRole.equals("admin")) {
            // Admin: Add/remove and position access (update limited to position)
            updBtn.setEnabled(true); // But restrict salary in update method
            delBtn.setEnabled(true);
            approveBtn.setEnabled(false); // Only super admin can approve
            salaryField.setEditable(false); // Can't change salary
        }
        // Super admin: All enabled (default)
    }

    private void addStaff() {
        if (!userRole.equals("super admin") && !userRole.equals("admin")) return;
        try {
            manager.addStaff(nameField.getText().trim(),
                    positionField.getText().trim(),
                    Double.parseDouble(salaryField.getText().trim()));
            refreshTables();
            clearFields();
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Invalid input");
        }
    }

    private void updateStaff() {
        if (!userRole.equals("super admin") && !userRole.equals("admin")) return;
        try {
            int id = Integer.parseInt(idField.getText());
            String name = nameField.getText();
            String position = positionField.getText();
            double salary = userRole.equals("admin") ? manager.findStaffById(id).getSalary() : Double.parseDouble(salaryField.getText()); // Admin can't change salary
            if (manager.updateStaff(id, name, position, salary)) {
                refreshTables();
                clearFields();
            }
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Update failed");
        }
    }

    private void deleteStaff() {
        if (!userRole.equals("super admin") && !userRole.equals("admin")) return;
        try {
            int id = Integer.parseInt(idField.getText());
            manager.deleteStaff(id);
            refreshTables();
            clearFields();
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Delete failed");
        }
    }

    private void searchStaff() {
        if (userRole.equals("user")) return;
        try {
            int id = Integer.parseInt(idField.getText());
            Staff s = manager.findStaffById(id);
            if (s != null) {
                nameField.setText(s.getName());
                positionField.setText(s.getPosition());
                salaryField.setText(String.valueOf(s.getSalary()));
            }
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Not found");
        }
    }

    private void markAttendance() {
        try {
            int id = userRole.equals("user") ? userId : Integer.parseInt(idField.getText());
            String status = JOptionPane.showInputDialog("Enter Present/Absent:");
            manager.markAttendance(id, LocalDate.now(), status);
            refreshTables();
        } catch (Exception ex) {}
    }

    private void submitLeaveRequest() {
        try {
            int id = userRole.equals("user") ? userId : Integer.parseInt(idField.getText());
            LocalDate start = LocalDate.parse(JOptionPane.showInputDialog("Start Date (YYYY-MM-DD):"));
            LocalDate end = LocalDate.parse(JOptionPane.showInputDialog("End Date (YYYY-MM-DD):"));
            String reason = JOptionPane.showInputDialog("Reason:");
            manager.addLeaveRequest(id, start, end, reason);
            refreshTables();
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Invalid input");
        }
    }

    private void approveLeaveRequest() {
        if (!userRole.equals("super admin")) return;
        try {
            int staffId = Integer.parseInt(JOptionPane.showInputDialog("Staff ID:"));
            int index = Integer.parseInt(JOptionPane.showInputDialog("Request Index (0-based):"));
            manager.approveLeaveRequest(staffId, index);
            refreshTables();
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Approval failed");
        }
    }

    private void refreshTables() {
        staffModel.setRowCount(0);
        attendanceModel.setRowCount(0);
        leaveModel.setRowCount(0);

        for (Staff s : manager.getAllStaff()) {
            if (userRole.equals("user") && s.getId() != userId) continue; // Users see only their own
            staffModel.addRow(new Object[]{s.getId(), s.getName(), s.getPosition(), s.getSalary()});
            s.getAttendance().forEach((date, status) ->
                    attendanceModel.addRow(new Object[]{date, status}));
            for (LeaveRequest req : s.getLeaveRequests()) {
                leaveModel.addRow(new Object[]{
                        req.getStartDate(), req.getEndDate(), req.getReason(), req.getStatus()});
            }
        }
    }

    private void clearFields() {
        if (!userRole.equals("user")) idField.setText("");
        nameField.setText("");
        positionField.setText("");
        salaryField.setText("");
    }
}

public class StaffManagementSystem {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new LoginSystem());
    }
}
