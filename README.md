 // https://mvnrepository.com/artifact/org.neo4j.driver/neo4j-java-driver
    implementation("org.neo4j.driver:neo4j-java-driver:5.28.4")

    // https://mvnrepository.com/artifact/org.projectlombok/lombok
    compileOnly("org.projectlombok:lombok:1.18.38")
    annotationProcessor("org.projectlombok:lombok:1.18.38")
import dao.DoctorDAO;
import dao.DoctorDAOImpl;
import modal.Doctor;
import modal.Patient;

import javax.print.Doc;
import java.io.DataInputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.List;
import java.util.Map;
import java.util.Scanner;

public class Server {

    public static void main(String[] args) throws IOException {
        try(ServerSocket serverSocket = new ServerSocket(8083)){
            System.out.println("Server started");
            while(true){
                Socket socket = serverSocket.accept();
                System.out.println(socket.getInetAddress());
                System.out.println(socket.getPort());

                Thread thread = new Thread( new HandlingClient(socket));
                thread.start();

            }
        }
    }
}

class HandlingClient implements Runnable{
    private Socket socket;
    private DoctorDAO doctorDAO;

    public HandlingClient(Socket socket) {
        this.socket = socket;
        doctorDAO = new DoctorDAOImpl();
    }

    @Override
    public void run() {
        try (ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
            DataInputStream in = new DataInputStream(socket.getInputStream());
            Scanner scanner = new Scanner(System.in)){

            while (true){
                String command = in.readUTF();

                switch (command){
                    case "FIND_DOCTOR"-> {
                        String doctorId = in.readUTF();

                        Doctor doctor = doctorDAO.findDoctorById(doctorId);
                        out.writeObject(doctor);
                        out.flush();
                    }
                    case "ADD_DOCTOR" ->{
                        String id = in.readUTF();
                        String name = in.readUTF();
                        String specialty = in.readUTF();
                        String phone = in.readUTF();

                        Doctor doctor = new Doctor(id, name, specialty, phone);
                        boolean result = doctorDAO.addDoctor(doctor);

                        out.writeBoolean(result);
                        out.flush();
                    }
                    case "GET_DOCTOR_COUNT" ->{
                        String departmentName = in.readUTF();

                        Map<String, Long> doctorCount = doctorDAO.getNoOfDoctorsBySpeciality(departmentName);
                        out.writeObject(doctorCount);
                        out.flush();
                    }
                    case "LIST_DOCTORS" ->{
                        String specialty = in.readUTF();

                        List<Doctor> doctorList = doctorDAO.listDoctorsBySpeciality(specialty);
                        out.writeObject(doctorList);
                        out.flush();
                    }
                    case "UPDATE_DIAGNOSIS" -> {
                        String doctorId = in.readUTF();
                        String patientId = in.readUTF();
                        String diagnosis = in.readUTF();

                        boolean result = doctorDAO.updateDiagnosis(patientId, doctorId, diagnosis);
                        out.writeBoolean(result);
                        out.flush();
                    }
                    case "TOTAL_PATIENTS" -> {
                        String doctorId = in.readUTF();

                        int totalPatients = doctorDAO.totalDoctorsByPatient(doctorId);
                        out.writeInt(totalPatients);
                        out.flush();
                    }
                    case "SEARCH_PATIENTS" ->{
                        String keyword = in.readUTF();

                        List<Patient> patientList = doctorDAO.fulltextSearchPatients(keyword);
                        out.writeObject(patientList);
                        out.flush();
                    }
                    case "COUNT_PATIENTS"->{
                        Map<Doctor,Long> countPatient = doctorDAO.countDoctorsByPatient();
                        out.writeObject(countPatient);
                        out.flush();
                    }
                    case "Exit" ->{
                        System.out.println("Client disconnected");
                        socket.close();
                        return;
                    }
                    default -> System.out.println("Invalid command received");
                }

            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
import modal.Doctor;
import modal.Patient;

import javax.print.Doc;
import java.io.DataOutputStream;
import java.io.ObjectInputStream;
import java.net.Socket;
import java.util.List;
import java.util.Map;
import java.util.Scanner;

public class Client {

    public static void main(String[] args) throws Exception {
        try (Socket socket = new Socket("localhost", 8083);
        DataOutputStream out = new DataOutputStream(socket.getOutputStream());
        ObjectInputStream in = new ObjectInputStream(socket.getInputStream());
        Scanner scanner = new Scanner(System.in);
        ){
            while (true) {
                System.out.println("\n===== Doctor Management System =====");
                System.out.println("1. Find Doctor by ID");
                System.out.println("2. Add New Doctor");
                System.out.println("3. Get Number of Doctors by Speciality and Department");
                System.out.println("4. List Doctors by Speciality");
                System.out.println("5. Update Patient Diagnosis");
                System.out.println("6. Total patient with Doctors by ID");
                System.out.println("7. Find Patient");
                System.out.println("8. Count Patient by Doctor");
                System.out.println("9. Exit");
                System.out.print("Choose an option: ");
                int choice = scanner.nextInt();
                scanner.nextLine(); // Consume newlines

                switch (choice) {
                    case 1 ->{
                        out.writeUTF("FIND_DOCTOR");
                        System.out.println("Enter Doctor ID: ");
                        String doctorID = scanner.nextLine();
                        out.writeUTF(doctorID);
                        out.flush();

                        Doctor doctor = (Doctor) in.readObject();
                        System.out.println(doctor);
                    }
                    case 2 -> {
                        out.writeUTF("ADD_DOCTOR");

                        System.out.println("Enter Doctor ID: ");
                        String doctorID = scanner.nextLine();
                        out.writeUTF(doctorID);

                        System.out.println("Enter Doctor Name: ");
                        String doctorName = scanner.nextLine();
                        out.writeUTF(doctorName);

                        System.out.println("Enter Doctor Speciality: ");
                        String speciality = scanner.nextLine();
                        out.writeUTF(speciality);

                        System.out.println("Enter Doctor Phone Number: ");
                        String phoneNumber = scanner.nextLine();
                        out.writeUTF(phoneNumber);
                        out.flush();

                        boolean isAdded = in.readBoolean();
                        if(isAdded){
                            System.out.println("Doctor Added");
                        }else{
                            System.out.println("Doctor Not Added");
                        }
                    }
                    case 3 ->{
                        out.writeUTF("GET_DOCTOR_COUNT");
                        System.out.println("Enter Department name: ");
                        String departmentName = scanner.nextLine();
                        out.writeUTF(departmentName);
                        out.flush();

                        Map<String, Long> doctorCount = (Map<String, Long>) in.readObject();
                        doctorCount.entrySet()
                                .forEach(entry-> System.out.println(entry.getKey() + ":"+ entry.getValue()));
                    }
                    case 4 ->{
                        out.writeUTF("LIST_DOCTORS");
                        System.out.println("Enter Speciality keyword: ");
                        String speciality = scanner.nextLine();
                        out.writeUTF(speciality);
                        out.flush();

                        List<Doctor> doctorList = (List<Doctor>) in.readObject();
                        doctorList.forEach(
                                doctor -> System.out.println(doctor)
                        );
                    }

                    case 5 ->{
                        out.writeUTF("UPDATE_DIAGNOSIS");
                        System.out.println("Enter Doctor ID: ");
                        String doctorID = scanner.nextLine();
                        out.writeUTF(doctorID);

                        System.out.println("Enter Patient ID");
                        String patientID = scanner.nextLine();
                        out.writeUTF(patientID);

                        System.out.println("Enter diagnosis");
                        String diagnosis = scanner.nextLine();
                        out.writeUTF(diagnosis);

                        out.flush();

                        boolean isUpdated = in.readBoolean();
                        if(isUpdated){
                            System.out.println("Doctor Updated");
                        }else {
                            System.out.println("Doctor Not Updated");
                        }
                    }
                    case 6 ->{
                        out.writeUTF("TOTAL_PATIENTS");
                        System.out.println("Enter Doctor ID: ");
                        String doctorID = scanner.nextLine();
                        out.writeUTF(doctorID);
                        out.flush();

                        int totalPatients = in.readInt();
                        System.out.println(totalPatients);
                    }
                    case 7 ->{
                        out.writeUTF("SEARCH_PATIENTS");

                        System.out.println("Enter keyword");
                        String keyword = scanner.nextLine();
                        out.writeUTF(keyword);
                        out.flush();

                        List<Patient> list = (List<Patient>) in.readObject();
                        list.forEach(
                                patient -> System.out.println(patient)
                        );
                    }
                    case 8->{
                        out.writeUTF("COUNT_PATIENTS");
                        out.flush();

                        Map<Doctor,Long> countPatients = (Map<Doctor,Long>) in.readObject();
                        countPatients.entrySet()
                                .forEach(entry -> System.out.println(entry.getKey()+" "+entry.getValue()));
                    }
                    case 9 ->{
                        System.out.println("Exiting....");
                        out.writeUTF("EXIT");
                        out.flush();
                        return;
                    }
                    default -> System.out.println("Invalid choice! Please try again.");
                }
            }
        }
    }
}
package util;

import modal.Doctor;
import modal.Gender;
import modal.Patient;
import org.neo4j.bolt.connection.AuthToken;
import org.neo4j.driver.*;
import org.neo4j.driver.types.Node;

import javax.print.Doc;
import java.util.Map;

public class AppUtils {
    public static final String DB_NAME = "Viet22671321";
    public static Driver getDriver(){
        String username = "neo4j";
        String password = "sapassword";
        String uri = "neo4j://localhost:7687";
        return GraphDatabase.driver(uri, AuthTokens.basic(username, password));
    }
    public static Session getSession(){
        return getDriver().session(SessionConfig.forDatabase(DB_NAME));
    }
    public static Doctor toDoctor(Node node){
        return new Doctor(
                node.get("doctor_id").asString(),
                node.get("name").asString(),
                node.get("phone").asString(),
                node.get("speciality").asString()
        );
    }
    public static Map<String, Object> toMap (Doctor doctor){
        return Map.of(
                "doctor_id", doctor.getId(),
                "name", doctor.getName(),
                "phone", doctor.getPhone(),
                "speciality", doctor.getSpeciality()
        );
    }

    public static Patient toPatient(Node node) {
        String genderStr = node.get("gender").asString().trim().toUpperCase();
        Gender gender = Gender.valueOf(genderStr);
        return new Patient(
                node.get("patient_id").asString(),
                node.get("name").asString(),
                node.get("phone").asString(),
                gender,
                node.get("date_of_birth").asLocalDate().toString(),
                node.get("address").asString()
        );
    }
}
package dao;

import modal.Doctor;
import modal.Patient;
import org.neo4j.driver.Query;
import org.neo4j.driver.Result;
import org.neo4j.driver.summary.ResultSummary;
import org.neo4j.driver.types.Node;
import util.AppUtils;

import java.util.List;
import java.util.Map;
import java.util.stream.Collector;
import java.util.stream.Collectors;

public class DoctorDAOImpl implements DoctorDAO {
    @Override
    public Doctor findDoctorById(String doctorId) {
        String query = "MATCH (d:Doctor {doctor_id: $doctorId})\n" +
                "RETURN d ";
        try (var session = AppUtils.getSession()) {
            return session.executeRead(tx->{
                Result result = tx.run(query, Map.of("doctorId", doctorId));
                if(result.hasNext()) {
                    var record = result.next();
                    Node node = record.get("d").asNode();
                    return AppUtils.toDoctor(node);
                }
                return null;
            });
        }
    }
    @Override
    public boolean addDoctor(Doctor doctor) {
        String query = "MERGE (d:Doctor {doctor_id:$doctor_id} )\n" +
                "SET d.name =$name,\n" +
                "d.phone =$phone,\n" +
                "d.speciality =$speciality\n" +
                "RETURN d";

        try(var session = AppUtils.getSession()) {
            return session.executeWrite(tx->{
                ResultSummary result = tx.run(query, AppUtils.toMap(doctor)).consume();
                return result.counters().nodesCreated() > 0;
            });
        }
    }
    @Override
    public Map<String, Long> getNoOfDoctorsBySpeciality(String deptName) {
        String query = "MATCH (d:Doctor) - [:BELONG_TO] -> (dep:Department{name:$name})\n" +
                "RETURN d.speciality as speciality, count(d) as noOfDoctors;";
        try(var session = AppUtils.getSession()) {
            return session.executeRead(tx->{
                Result result = tx.run(query, Map.of("name", deptName));

                return result.stream()
                        .collect(Collectors.toMap(
                                record -> record.get("speciality").asString(),
                                record -> record.get("noOfDoctors").asLong()
                        ));
            });
        }
    }

    @Override
    public List<Doctor> listDoctorsBySpeciality(String keyword) {
        try (var session = AppUtils.getSession()){
            return session.executeRead(tx->{
                String query = "CALL db.index.fulltext.queryNodes(\"listDoctorsBySpeciality\",$keyword) YIELD node\n" +
                        "RETURN node";
                Result result = tx.run(query, Map.of("keyword", keyword));
                if(!result.hasNext()) {
                    return null;
                }
                return result.stream()
                        .map(record -> record.get("node").asNode())
                        .map(node -> AppUtils.toDoctor(node))
                        .collect(Collectors.toList());
            });
        }
    }
    @Override
    public int totalDoctorsByPatient(String doctorId) {
        try (var session = AppUtils.getSession()){
            return session.executeRead(tx->{
                String query = "MATCH (d:Doctor {doctor_id:$doctorId}) <- [:BE_TREATED] - (p:Patient)\n" +
                        "RETURN count(p) as total";
                Result result = tx.run(query, Map.of("doctorId", doctorId));
                if(!result.hasNext()) {
                    return 0;
                }
                return result.single().get("total").asInt();
            });
        }
    }
    @Override
    public boolean updateDiagnosis(String patientId, String doctorId, String newDiagnosis) {
        try (var session = AppUtils.getSession()){
            return session.executeWrite(tx->{
                String query = "MATCH (p:Patient {patient_id:$patient_id}) -[r:BE_TREATED] -> (d:Doctor {doctor_id:$doctor_id})\n" +
                        "WHERE r.end_date IS NULL\n" +
                        "SET r.diagnosis =$diagnosis";

                Map<String, Object> map = Map.of("patient_id", patientId,
                        "doctor_id", doctorId,
                        "diagnosis", newDiagnosis);

                ResultSummary resultSummary = tx.run(query,map).consume();
                return resultSummary.counters().propertiesSet() > 0;
            });
        }
    }

    @Override
    public List<Patient> fulltextSearchPatients(String keyword) {
        try(var session = AppUtils.getSession()){
            return session.executeRead(tx->{
                String query = "CALL db.index.fulltext.queryNodes(\"fullTextSearchPatient\",$keyword) YIELD node\n" +
                        "RETURN node";
                Result result = tx.run(query,Map.of("keyword", keyword));
                if(!result.hasNext()) {
                    return null;
                }
                return result.stream()
                        .map(record -> record.get("node").asNode())
                        .map(node -> AppUtils.toPatient(node))
                        .collect(Collectors.toList());
            });
        }
    }

    @Override
    public Map<Doctor, Long> countDoctorsByPatient() {
        try(var session  = AppUtils.getSession()){
            return session.executeRead(tx->{
               String query = "MATCH (p:Patient) - [:BE_TREATED] -> (d:Doctor)\n" +
                       "RETURN d, count(p) as numOfPatient";

               Result result = tx.run(query);
               if(!result.hasNext()) {
                   return null;
               }
               return result.stream()
                       .collect(Collectors.toMap(
                               record -> AppUtils.toDoctor(record.get("d").asNode()),
                               record -> record.get("numOfPatient").asLong()
                       ));
            });
        }
    }
}
package modal;

import lombok.*;

import java.io.Serializable;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@ToString(callSuper = true)
public class Doctor extends Person implements Serializable {
    private String speciality;
    @ToString.Exclude
    private transient Department department;

    public Doctor(String id, String name, String phone, String speciality) {
        super(id, name, phone);
        this.speciality = speciality;
    }
}
package modal;

import lombok.AllArgsConstructor;
import lombok.ToString;

@ToString
@AllArgsConstructor
public enum Gender {
    MALE, FEMALE, ORTHER
}
import dao.DoctorDAO;
import dao.DoctorDAOImpl;
import modal.Doctor;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;


@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class DoctorDAOTest {
    private DoctorDAO dao;
    @BeforeAll
    void setUp(){
        dao = new DoctorDAOImpl();
    }

    @Test
    void findDoctorByIDTest(){
        Doctor doctor = dao.findDoctorById("DR.010");
        assertEquals("Daniel Rodriguez", doctor.getName());
        assertEquals("0567.890.123", doctor.getPhone());
        assertEquals("Ophthalmology and Optometry", doctor.getSpeciality());
    }

//    @Test
//    void addDoctorTest(){
//        Doctor doctor = new Doctor("DR.1114", "Doctor Test", "1234.567.890", "Cardiology");
//        boolean result = dao.addDoctor(doctor);
//        assertEquals(true, result);
//    }

    @Test
    void getNoOfDoctorsBySpecialityTest(){
        String deptName = "Internal Medicine";
        Map<String, Long> result = dao.getNoOfDoctorsBySpeciality(deptName);
        assertEquals(5, result.size());
        assertEquals(2L, result.get("Internal Medicine"));
        assertEquals(1L, result.get("Dermatology Services"));
    }
    @Test
    void listDoctorsBySpecialityTest(){
        String keyName = "Medicine";
        List<Doctor> result = dao.listDoctorsBySpeciality(keyName);
        assertEquals(11, result.size());
        assertEquals("DR.005", result.get(0).getId());
        assertEquals("DR.024", result.get(1).getId());
    }

    @Test
    void updateDiagnosis(){
        String diagnosis = "Test DAO updateDiagnosis";
        boolean result = dao.updateDiagnosis("PT006","DR.1111",diagnosis);
        assertEquals(true, result);
    }
    @Test
    void TotalDoctorsByPatientTest(){
        String doctorId = "DR.018";
        int result = dao.totalDoctorsByPatient(doctorId);
        assertEquals(2, result);
    }
    @AfterAll
    void tearDown(){
        dao = null;
    }
}
