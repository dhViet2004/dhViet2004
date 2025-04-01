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
