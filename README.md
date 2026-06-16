# Lab_Notes

-- susunan file --
1. package model --> file student.java (javabeans)
2. package dao --> file dao --> file studentDAO.java
3. package controller --> file studentServlet.java (utk servlet)
4. web pages --> file student-form.jsp & student-list.jsp (jsp file kt sini)

**STEP 1: MODEL COMPONENT (JavaBeans)
Fail: src/java/model/Student.java Jenis Komen: Struktur Anatomi JavaBeans (Encapsulation)
package model;**

// FUNGSIONAL: Wajib import Serializable untuk membolehkan 'state' objek disimpan/dipindahkan
import java.io.Serializable;

/**
 * JENIS: JavaBeans / Model
 * FUNGSI: Sebagai kontena mudah alih untuk memegang data pelajar antara komponen MVC.
 */
public class Student implements Serializable {
    
    // ==========================================
    // 1. INSTANCE VARIABLES (Ciri Encapsulation)
    // ==========================================
    // PERATURAN: Mesti PRIVATE supaya data tidak boleh diakses/diubah secara terus dari luar kelas.
    private String id;
    private String name;
    private String program;

    // ==========================================
    // 2. DEFAULT CONSTRUCTOR (No-Argument)
    // ==========================================
    // PERATURAN: Wajib ada public constructor tanpa parameter supaya web container boleh cipta objek ini secara dinamik.
    public Student() {
    }

    // ==========================================
    // 3. GETTER & SETTER METHODS
    // ==========================================
    // FUNGSI GETTER: Untuk membolehkan komponen lain (seperti JSP/EL) MEMBACA nilai variable private.
    // FUNGSI SETTER: Untuk membolehkan komponen lain (seperti Servlet) MENULIS/MENUKAR nilai variable private.
    
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id; // 'this.id' merujuk kepada variable private di atas, 'id' merujuk kepada parameter fungsi
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getProgram() {
        return program;
    }

    public void setProgram(String program) {
        this.program = program;
    }
}


**STEP 2: DATA ACCESS OBJECT (DAO)
Fail: src/java/dao/StudentDAO.java Jenis Komen: Logik JDBC, SQL Execution, dan Pengurusan Sumber (Database)**

package dao;

// FUNGSIONAL: Import semua library SQL utama untuk berinteraksi dengan MySQL
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import model.Student; // Import model Student untuk digunakan dalam senarai/data

/**
 * JENIS: Data Access Object (DAO)
 * FUNGSI: Memusatkan semua operasi SQL (CRUD). Servlet tidak akan tahu apa-apa pasal kod SQL, ia hanya panggil kelas ini.
 */
public class StudentDAO {

    // =========================================================
    // FUNGSI UTAMA: PEMBINAAN SAMBUNGAN DATABASE (CONNECTION)
    // =========================================================
    // Kategori: Private Utility Method
    private Connection getConnection() {
        Connection conn = null;
        try {
            // LANGKAH 1 (Rujuk Nota 4): Memuatkan pemandu (Driver) MySQL ke dalam memori aplikasi
            Class.forName("com.mysql.cj.jdbc.Driver");
            
            // LANGKAH 2: Menetapkan maklumat URL, Username, dan Password database makmal
            String url = "jdbc:mysql://localhost:3306/studentdb"; // Pastikan nama DB 'studentdb' wujud di MySQL
            String user = "root";
            String password = ""; // Jika komputer lab ada password, isi di sini
            
            // LANGKAH 3: Membuka pintu gerbang sambungan
            conn = DriverManager.getConnection(url, user, password);
        } catch (ClassNotFoundException | SQLException e) {
            e.printStackTrace(); // Paparkan ralat pada log pelayan jika pemandu/pautan gagal
        }
        return conn; // Memulangkan objek sambungan yang aktif
    }

    // =========================================================
    // [C] - CREATE: MENAMBAH REKOD BARU (INSERT)
    // =========================================================
    public boolean insertStudent(Student student) {
        String sql = "INSERT INTO students (id, name, program) VALUES (?, ?, ?)";
        
        // TEKNIK: try-with-resources. Ia automatik menutup 'conn' dan 'ps' selepas tamat untuk elak kebocoran memori (Memory Leak).
        try (Connection conn = this.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            // FUNGSI: Mengikat data dari JavaBeans ke dalam tanda soal (?) SQL secara selamat (Elak SQL Injection)
            ps.setString(1, student.getId());      // Tanda soal pertama (1) diisi dengan ID
            ps.setString(2, student.getName());    // Tanda soal kedua (2) diisi dengan Nama
            ps.setString(3, student.getProgram()); // Tanda soal ketiga (3) diisi dengan Program
            
            // REKOD DATA: executeUpdate() digunakan untuk INSERT, UPDATE, DELETE. Ia memulangkan jumlah baris yang berubah.
            int rowsAffected = ps.executeUpdate(); 
            return rowsAffected > 0; // Pulang 'true' jika ada baris berjaya dimasukkan (maknanya > 0)
            
        } catch (SQLException e) {
            e.printStackTrace();
            return false;
        }
    }

    // =========================================================
    // [R] - READ: MENGAMBIL SEMUA REKOD (SELECT ALL)
    // =========================================================
    public List<Student> selectAllStudents() {
        List<Student> students = new ArrayList<>(); // Cipta senarai array kosong untuk simpan semua pelajar
        String sql = "SELECT * FROM students";
        
        try (Connection conn = this.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql);
             // PAPAR DATA (Rujuk Nota 4): Khas kueri SELECT, wajib guna executeQuery(). Ia memulangkan objek ResultSet (jadual data virtual).
             ResultSet rs = ps.executeQuery()) { 
            
            // GELUNGAN: Selagi ada baris data seterusnya dalam ResultSet (rs.next()), teruskan membaca
            while (rs.next()) {
                Student student = new Student(); // Cipta objek bean baru untuk baris semasa
                
                // FUNGSI: Ekstrak data dari kolum MySQL berdasarkan nama kolum, masukkan ke dalam Bean guna SETTER
                student.setId(rs.getString("id"));
                student.setName(rs.getString("name"));
                student.setProgram(rs.getString("program"));
                
                students.add(student); // Masukkan objek bean pelajar yang lengkap ke dalam senarai Array
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return students; // Pulangkan senarai pelajar (boleh jadi kosong atau berisi)
    }

    // =========================================================
    // [R] - READ: MENGAMBIL SATU REKOD UTK EDIT FORM (SELECT BY ID)
    // =========================================================
    public Student selectStudentById(String id) {
        Student student = null;
        String sql = "SELECT * FROM students WHERE id = ?";
        
        try (Connection conn = this.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            ps.setString(1, id); // Ikat ID parameter ke tanda soal pertama
            
            try (ResultSet rs = ps.executeQuery()) {
                // LOGIK: Guna 'if' bukan 'while' sebab kita jangka hanya ada SATU atau TIADA rekod dipulangkan (ID adalah Primary Key)
                if (rs.next()) {
                    student = new Student();
                    student.setId(rs.getString("id"));
                    student.setName(rs.getString("name"));
                    student.setProgram(rs.getString("program"));
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return student; // Pulang null jika pelajar tidak wujud, atau pulang objek jika wujud
    }

    // =========================================================
    // [U] - UPDATE: MENGEMASKINI DATA (UPDATE)
    // =========================================================
    public boolean updateStudent(Student student) {
        String sql = "UPDATE students SET name = ?, program = ? WHERE id = ?";
        
        try (Connection conn = this.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            ps.setString(1, student.getName());    // ? pertama
            ps.setString(2, student.getProgram()); // ? kedua
            ps.setString(3, student.getId());      // ? ketiga (WHERE clause)
            
            return ps.executeUpdate() > 0; // Guna executeUpdate() kerana ia menukar struktur data asal
        } catch (SQLException e) {
            e.printStackTrace();
            return false;
        }
    }

    // =========================================================
    // [D] - DELETE: MEMADAM DATA (DELETE)
    // =========================================================
    public boolean deleteStudent(String id) {
        String sql = "DELETE FROM students WHERE id = ?";
        
        try (Connection conn = this.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            ps.setString(1, id);
            return ps.executeUpdate() > 0; // Guna executeUpdate() kerana melibatkan pemadaman rekod
        } catch (SQLException e) {
            e.printStackTrace();
            return false;
        }
    }
}


**STEP 3: CONTROLLER COMPONENT (Servlet)
Fail: src/java/controller/StudentServlet.java Jenis Komen: Kitaran Hayat Servlet, Traffic Routing, Scope Management, dan Borang Request Handling**

package controller;

import dao.StudentDAO;
import java.io.IOException;
import java.util.List;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import model.Student;

/**
 * JENIS: Controller / Servlet
 * FUNGSI: Pengendali aliran trafik sistem. Ia membaca arahan pengguna, bercakap dengan DAO, dan menghantar data ke page JSP.
 */
@WebServlet("/StudentServlet") // URL MAPPING PERATURAN: Mesti bermula dengan tanda '/' dan sama ejaan dengan action form JSP
public class StudentServlet extends HttpServlet {
    
    // Global variable untuk DAO supaya semua method dalam kelas ini boleh menggunakannya
    private StudentDAO studentDAO;

    // ==========================================
    // 1. SERVLET LIFECYCLE: init() METHOD
    // ==========================================
    // PERATURAN (Rujuk Nota 2): Method ini jalan SEKALI SAHAJA sepanjang hayat Servlet dijalankan.
    // FUNGSI: Paling sesuai untuk buat persediaan/initialization objek berat seperti DAO.
    @Override
    public void init() {
        studentDAO = new StudentDAO();
    }

    // ==========================================
    // 2. REQUEST HANDLER: doGet() METHOD
    // ==========================================
    // FUNGSI (Rujuk Nota 3): Menguruskan semua request berjenis GET (Contoh: Klik hyperlink <a>, klik butang padam, atau taip URL terus di browser).
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        
        // Ambil string parameter bernama 'action' dari URL (Contoh: StudentServlet?action=delete)
        String action = request.getParameter("action");
        if (action == null) {
            action = "list"; // Jika pengguna cuma buka link biasa tanpa letak parameter, set default aksi sebagai paparan senarai (list)
        }

        try {
            // ROUTING SWITCH: Menghantar trafik ke fungsi yang betul berdasarkan nilai parameter 'action'
            switch (action) {
                case "delete":
                    deleteStudent(request, response);
                    break;
                case "edit":
                    showEditForm(request, response);
                    break;
                default:
                    listStudents(request, response);
                    break;
            }
        } catch (Exception e) {
            throw new ServletException(e);
        }
    }

    // ==========================================
    // 3. REQUEST HANDLER: doPost() METHOD
    // ==========================================
    // FUNGSI (Rujuk Nota 3): Menguruskan penghantaran borang selamat berjenis POST (Data disorok dalam HTTP Body). Contoh: Klik butang Submit form.
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        
        // Membaca 'hidden input' bernama action dari borang untuk tahu sama ada user tengah menekan borang 'insert' atau borang 'update'
        String action = request.getParameter("action");
        
        try {
            if ("update".equals(action)) {
                updateStudent(request, response);
            } else {
                insertStudent(request, response);
            }
        } catch (Exception e) {
            throw new ServletException(e);
        }
    }

    // ==========================================
    // 4. CORE OPERATIONAL METHODS (Logik Trafik)
    // ==========================================
    
    // FUNGSI: Ambil semua senarai pelajar dari DB dan forward ke list JSP
    private void listStudents(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        List<Student> listStudent = studentDAO.selectAllStudents();
        
        // STATE MANAGEMENT (Rujuk Nota 3): Menggunakan REQUEST SCOPE (Jangka hayat pendek) untuk simpan senarai objek.
        // "listStudent" adalah Key (nama samaran untuk dipanggil di JSP), listStudent adalah Value (objek senarai)
        request.setAttribute("listStudent", listStudent);
        
        // ROUTING TRAFIK: Menggunakan RequestDispatcher.forward() untuk gerakkan kawalan berserta data ke halaman student-list.jsp
        request.getRequestDispatcher("student-list.jsp").forward(request, response);
    }

    // FUNGSI: Ambil input borang pendaftaran, hantar ke DAO untuk simpan rekod baru
    private void insertStudent(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        // Cipta objek bean baru untuk sumbat data borang
        Student student = new Student();
        
        // PERATURAN PARAMETER: Ejaan string dalam getParameter("...") MESTI sama kes dengan attribute 'name' pada tag <input> HTML/JSP
        student.setId(request.getParameter("id"));
        student.setName(request.getParameter("name"));
        student.setProgram(request.getParameter("program"));

        studentDAO.insertStudent(student); // Panggil DAO untuk buat SQL INSERT
        
        // STRATEGI POST-REDIRECT-GET: Guna response.sendRedirect() selepas operasi simpan (POST).
        // Kenapa? Supaya URL browser bertukar bersih. Kalau user refresh page, dia tak akan hantar data duplikasi/simpan dua kali (Rujuk Nota 2).
        response.sendRedirect("StudentServlet?action=list");
    }

    // FUNGSI: Ambil ID khusus dari link, dapatkan objek data pelajar penuh dari DB, sumbat dalam request, buka borang edit
    private void showEditForm(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        String id = request.getParameter("id"); // Ambil id pelajar yang hendak di-edit dari klik link
        Student existingStudent = studentDAO.selectStudentById(id); // Cari data penuh dari DB guna ID tersebut
        
        request.setAttribute("student", existingStudent); // Simpan objek pelajar sedia ada dalam request scope
        request.getRequestDispatcher("student-form.jsp").forward(request, response); // Buka borang student-form.jsp (Borang akan auto-terisi data lama)
    }

    // FUNGSI: Ambil data baru yang diubah suai dari borang edit, panggil DAO untuk buat SQL UPDATE
    private void updateStudent(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        Student student = new Student();
        student.setId(request.getParameter("id"));
        student.setName(request.getParameter("name"));
        student.setProgram(request.getParameter("program"));

        studentDAO.updateStudent(student); // Panggil DAO untuk kemas kini data dalam pangkalan data MySQL
        response.sendRedirect("StudentServlet?action=list"); // Kembali ke senarai utama yang dikemas kini
    }

    // FUNGSI: Ambil parameter ID dari klik link, panggil DAO untuk padam dari database
    private void deleteStudent(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        String id = request.getParameter("id"); // Ambil ID pelajar yang hendak dibuang
        studentDAO.deleteStudent(id); // Panggil DAO untuk buat SQL DELETE
        response.sendRedirect("StudentServlet?action=list"); // Refresh senarai paparan utama
    }
}


**STEP 4: VIEW COMPONENT (Borang Dinamik)
Fail: Web Pages/student-form.jsp Jenis Komen: JSTL Core Tags, Expression Language (EL), Dinamik Layout Form**

<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@taglib uri="http://java.sun.com/jsp/libs/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head>
    <title>Borang Maklumat Pelajar</title>
</head>
<body>
    <h2>
        <c:if test="${student != null}">Kemaskini Maklumat Pelajar</c:if>
        <c:if test="${student == null}">Daftar Pelajar Baru</c:if>
    </h2>

    <form action="StudentServlet" method="POST">
        
        <input type="hidden" name="action" value="${student != null ? 'update' : 'insert'}">

        ID Pelajar: 
        <input type="text" name="id" value="${student.id}" ${student != null ? 'readonly' : ''} required><br><br>

        Nama Penuh: 
        <input type="text" name="name" value="${student.name}" required><br><br>

        Program Pengajian: 
        <input type="text" name="program" value="${student.program}" required><br><br>

        <input type="submit" value="${student != null ? 'Kemaskini Data' : 'Daftar Sekarang'}">
    </form>
    <br>
    <a href="StudentServlet?action=list">Lihat Senarai Pelajar</a>
</body>
</html>


**STEP 5: VIEW COMPONENT (Jadual Senarai Rekod)
Fail: Web Pages/student-list.jsp Jenis Komen: JSTL Looping Iterator, UI Render Data, URL Query Data Handling**

<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@taglib uri="http://java.sun.com/jsp/libs/core" prefix="c"%>
<!DOCTYPE html>
<html>
<head>
    <title>Sistem Pelajar cse3023</title>
</head>
<body>
    <h2>Senarai Keseluruhan Pelajar (Database)</h2>
    <a href="student-form.jsp">Tambah Pelajar Baru</a>
    <br><br>

    <table border="1" cellpadding="7">
        <thead>
            <tr>
                <th>ID</th>
                <th>Nama Penuh</th>
                <th>Program</th>
                <th>Tindakan Pembersihan / Kemaskini</th>
            </tr>
        </thead>
        <tbody>
            <c:forEach var="stud" items="${listStudent}">
                <tr>
                    <td>${stud.id}</td>
                    <td>${stud.name}</td>
                    <td>${stud.program}</td>
                    <td>
                        <a href="StudentServlet?action=edit&id=${stud.id}">Edit</a> | 
                        
                        <a href="StudentServlet?action=delete&id=${stud.id}" 
                           onclick="return confirm('Adakah anda pasti untuk memadam data ini?');">Padam</a>
                    </td>
                </tr>
            </c:forEach>
        </tbody>
    </table>
</body>
</html>








---- error detection & mcm mana nak tau kt mana salah ----

1. HTTP Error 404: Not Found (Fail Tidak Dijumpai)
Di mana nak cari salah (Checklist 404):
-- Arah Laluan Borang (Form Action): Periksa fail JSP input awak. Jika awak tulis <form action="StudentServlet">, periksa pula di dalam fail Servlet awak. Adakah annotation di atas kelas Servlet ditulis tepat @WebServlet("/StudentServlet")?

-- Perangkap popular: Tertinggal tanda / dalam @WebServlet atau salah huruf besar/kecil (Kes sensitif).

- Ejaan Nama Fail JSP: Periksa dalam Servlet semasa awak buat forwarding data:
  request.getRequestDispatcher("student-list.jsp").forward(request, response);

  Adakah fail tersebut betul-betul bernama student-list.jsp dan diletakkan di dalam folder Web     Pages (bukan di dalam WEB-INF)? Jika ejaan salah walaupun satu huruf, ralat 404 akan keluar.



2. HTTP Error 500: Internal Server Error (Kod Java Rosak/Crash)
   Maksud: Server jumpa fail Servlet atau JSP tersebut, tetapi semasa server cuba menjalankan kod  Java di dalamnya, kod itu terhenti (crash) di tengah jalan akibat ralat logik atau ralat sistem.

Bagaimana cara tahu salah kat mana? (Teknik Jejak Ralat 500)
-- Apabila skrin browser bertukar menjadi putih/kelabu dan memaparkan teks ralat 500 yang panjang, jangan panik. Sila lihat dua tempat ini:

i) Tempat A: Paparan "Stack Trace" di BrowserTeks tulisan kecil yang panjang di browser itu dipanggil Stack Trace. Jangan baca dari atas, sebaliknya skrol ke bawah dan cari perkataan Caused by: atau cari baris kod yang menyebut nama package projek awak **(contoh: dao.StudentDAO atau controller.StudentServlet)**.

ii) Di hujung ayat tersebut, ia akan memaparkan nombor baris fail yang rosak.
Contoh teks: **at dao.StudentDAO.insertStudent(StudentDAO.java:45)** --> Ini bermakna kod Java awak crash tepat **pada baris ke-45 di dalam fail StudentDAO.java**. Buka fail tersebut dan tengok apa ada di baris 45.

iii) Tempat B: Tab "Output" atau "Console" dalam IDE (NetBeans/Eclipse)
Kadang-kadang browser tidak memaparkan perincian ralat penuh. Anda wajib melihat ruangan Output / Apache Tomcat Log di bahagian bawah IDE awak semasa ralat 500 itu berlaku. Di situ akan tertera tulisan berwarna merah yang memberitahu jenis ralatnya.

Jenis Ralat 500 yang Paling Popular dalam Lab Test:

1. **NullPointerException** Sebab: Kod awak cuba membaca sesuatu objek yang kosong (null).Punca biasa: Ejaan **name="..."** pada tag **<input>** di JSP **tidak sama** dengan string di dalam **request.getParameter("...") di Servlet**. Akibatnya, Servlet mengambil data kosong.
2. **SQLSyntaxErrorException** atau **MySQLSyntaxErrorException** Sebab: Ayat SQL awak salah tatabahasa.
Punca biasa: Salah ejaan nama jadual (table), salah ejaan nama kolum (column), atau **bilangan tanda soal ? dalam PreparedStatement tidak cukup/terlebih** berbanding data yang diikat **(ps.setString)**.
3. **ClassNotFoundException: com.mysql.cj.jdbc.Driver** Sebab: Kod Java tidak kenal di mana pemandu (Driver) MySQL berada.
Punca biasa: Awak lupa masukkan fail JAR MySQL Connector ke dalam folder Libraries projek awak 
