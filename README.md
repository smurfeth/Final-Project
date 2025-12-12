using System;
using System.Collections.Generic;
using System.Linq;
using System.IO;
using ConsoleTables;
using Spectre.Console;

namespace CATADMAN_final
{
    public interface Imanage
    {
        void Add();
        void Update();
        void Delete();
        void View();
    }
    public abstract class User
    {
        private string Username;
        private string Password;
        private string FullName;
        public string username { get { return Username; } set { Username = value; } }
        public string password { get { return Password; } }
        public string fullname { get { return FullName; } set { FullName = value; } }

        public User(string username, string password, string fullname)
        {
            this.Username = username;
            this.Password = password;
            this.FullName = fullname;
        }
        public virtual void Login() { Console.WriteLine($"Welcome, {FullName}!"); }
        public virtual void Logout() { Console.WriteLine($"{FullName} logged out successfully."); }
        public abstract void DisplayMenu();
    }

    public class Receptionist : User
    {
        public string StaffID;
        public Receptionist(string username, string password, string fullName, string id)
            : base(username, password, fullName)
        {
            StaffID = id;
        }

        public override void DisplayMenu()
        {
            bool running = true;
            while (running)
            {
                Console.WriteLine($"\n---------- STAFF MENU ----------");
                Console.WriteLine("[1] View Walk-in & Upcoming Appointments Queue");
                Console.WriteLine("[2] View All Appointments");
                Console.WriteLine("[3] Serve Next Patient");
                Console.WriteLine("[4] Skip Walk-in Patient");
                Console.WriteLine("[5] Assign Walk-in");
                Console.WriteLine("[6] Reset Appointments and Queue (Archive)");
                Console.WriteLine("[7] Logout");
                Console.Write("Enter Choice: ");

                if (!int.TryParse(Console.ReadLine(), out int choice)) choice = -1;

                switch (choice)
                {
                    case 1:
                        ViewQueueWithAppointments();
                        break;
                    case 2:
                        Program.ViewAllAppointments();
                        break;
                    case 3:
                        Program.Queue.ServeNextPatient();
                        break;
                    case 4:
                        Program.Queue.SkipPatient();
                        break;
                    case 5:
                        Program.ManageCalendar(this);
                        break;
                    case 6:
                        Program.ArchiveAndResetAll();
                        break;
                    case 7:
                        running = false;
                        Logout();
                        break;
                    default:
                        Console.WriteLine("Invalid choice. Try again.");
                        break;
                }
            }
        }

        private void ViewQueueWithAppointments()
        {
            var today = DateTime.Today.Date;
            var allAppointments = Program.Appointments;

            var upcomingAppointments = allAppointments
                .Where(a => a.Type == "APPOINTMENT" && a.Date.Date == today)
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .ToList();

            List<dynamic> walkinQueue = Program.Queue.WaitingList.Select(w =>
            {
                var walkinApp = allAppointments
                    .FirstOrDefault(a => a.Type == "WALKIN" && a.PatientName.Equals(w.fullname, StringComparison.OrdinalIgnoreCase) && a.Date.Date == today);

                return (dynamic)new
                {
                    Patient = w,
                    Appointment = walkinApp
                };
            })
                .Where(item => item.Appointment != null)
                .OrderBy(item => item.Appointment.Date)
                .ThenBy(item => TimeSpan.Parse(item.Appointment.Time))
                .ThenBy(item => Program.GetQueueArrivalIndex(item.Patient))
                .ToList();

            DisplayCombinedServiceOrder(upcomingAppointments, walkinQueue);
        }
        private void DisplayCombinedServiceOrder(List<Appointment> upcomingAppointments, List<dynamic> walkinQueue)
        {
            var combinedList = new List<dynamic>();

            foreach (var a in upcomingAppointments)
            {
                var p = Program.GetPatientByName(a.PatientName) ?? new Patient(a.PatientName, "", a.PatientName, 0, "", "", "", "");
                combinedList.Add(new
                {
                    Patient = p,
                    Appointment = a,
                    TypeDisplay = "Appointment",
                    ServiceTime = a.Date.Date.Add(TimeSpan.Parse(a.Time))
                });
            }

            foreach (var item in walkinQueue)
            {
                combinedList.Add(new
                {
                    Patient = item.Patient,
                    Appointment = item.Appointment,
                    TypeDisplay = "Walk-in",
                    ServiceTime = item.Appointment.Date.Date.Add(TimeSpan.Parse(item.Appointment.Time))
                });
            }

            var finalServiceOrder = combinedList
                .OrderBy(c => c.ServiceTime)
                .ToList();

            Console.WriteLine("\n--- SERVICE ORDER QUEUE ---");
            if (finalServiceOrder.Count == 0)
            {
                Console.WriteLine("No patients in the queue or with assigned appointments.");
                return;
            }

            var tableCombined = new ConsoleTable("No.", "Patient Name", "Type", "Purpose", "Date", "Time");
            for (int i = 0; i < finalServiceOrder.Count; i++)
            {
                var c = finalServiceOrder[i];
                tableCombined.AddRow(
                    i + 1,
                    c.Patient.fullname,
                    c.TypeDisplay,
                    c.Appointment?.Purpose ?? "",
                    c.Appointment?.Date.ToString("MMM dd, yyyy") ?? "",
                    c.Appointment?.Time ?? ""
                );
            }
            tableCombined.Write();
        }
    }

    public class Patient : User
    {
        public int Age { get; set; }
        public string Contact { get; set; }
        public string Address { get; set; }
        public string ContactPerson { get; set; }
        public string Contactno { get; set; }

        public List<Appointment> MyAppointments = new List<Appointment>();

        public Patient(string username, string password, string fullName, int age, string contact, string address, string contactPerson, string contactNo)
            : base(username, password, fullName)
        {
            Age = age;
            Contact = contact;
            Address = address;
            ContactPerson = contactPerson;
            Contactno = contactNo;
        }
        public override void DisplayMenu()
        {
            bool running = true;
            while (running)
            {
                Console.WriteLine($"\n---------- PATIENT MENU ({fullname}) ----------");
                Console.WriteLine("[1] Book Appointment");
                Console.WriteLine("[2] View Appointments");
                Console.WriteLine("[3] Cancel Appointment");
                Console.WriteLine("[4] View Medical History");
                Console.WriteLine("[5] Logout");
                Console.Write("Enter choice: ");
                if (!int.TryParse(Console.ReadLine(), out int Choice)) Choice = -1;

                switch (Choice)
                {
                    case 1:
                        BookAppointment();
                        break;
                    case 2:
                        ViewAppointment();
                        break;
                    case 3:
                        CancelAppointment();
                        break;
                    case 4:
                        MedicalHistoryManager.ViewHistory(fullname);
                        break;
                    case 5:
                        running = false;
                        Logout();
                        break;
                    default:
                        Console.WriteLine("Invalid choice! Try again.");
                        break;
                }
            }
        }
        public void BookAppointment()
        {
            Console.Write("\nEnter Purpose of Visit: ");
            string purpose = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(purpose))
            {
                Console.WriteLine("Purpose of visit cannot be empty. Booking cancelled.");
                return;
            }

            DateTime today = DateTime.Today;
            var appointments = Program.Appointments;
            var freeSlotsByDate = new Dictionary<DateTime, List<string>>();

            Console.Clear();
            Console.WriteLine($"\n=== Weekly Appointment Calendar ===");
            Console.Write("Time".PadRight(8));
            for (int d = 0; d < 7; d++)
            {
                DateTime day = today.AddDays(d);
                Console.Write(day.ToString("MM-dd").PadRight(12));
                freeSlotsByDate[day] = new List<string>();
            }
            Console.WriteLine();

            int slotIndex = 1;
            var availableSlotsMap = new Dictionary<int, (DateTime date, string time)>();

            foreach (string slot in Program.TimeSlots)
            {
                Console.Write(slot.PadRight(8));
                for (int d = 0; d < 7; d++)
                {
                    DateTime day = today.AddDays(d);

                    bool occupied = appointments.Any(a =>
                        a.Date.Date == day.Date && a.Time == slot && a.Type != "FREE"
                    );

                    bool blocked = appointments.Any(a =>
                        a.Date.Date == day.Date && a.Time == slot && a.Type == "BUSY"
                    );

                    Console.ForegroundColor = blocked ? ConsoleColor.Yellow : (occupied ? ConsoleColor.Red : ConsoleColor.Green);

                    string mark;
                    if (blocked)
                    {
                        mark = "[B]";
                    }
                    else if (occupied)
                    {
                        mark = "[X]";
                    }
                    else
                    {
                        mark = $"[{slotIndex}]";
                        availableSlotsMap[slotIndex] = (day, slot);
                        slotIndex++;
                    }

                    Console.Write(mark.PadRight(12));
                    Console.ResetColor();
                }
                Console.WriteLine();
            }
            Console.WriteLine("\nLegend: [N] Available | [X] Booked ");

            if (availableSlotsMap.Count == 0)
            {
                Console.WriteLine("\nNo available slots for booking in the next week.");
                Console.WriteLine("Press any key to return...");
                Console.ReadKey();
                return;
            }

            Console.Write("\nEnter the [GREEN]number[/] of the slot you want to book: ");
            if (!int.TryParse(Console.ReadLine(), out int choiceNum) || !availableSlotsMap.ContainsKey(choiceNum))
            {
                Console.WriteLine("Invalid selection or slot is not available. Booking cancelled.");
                Console.ReadKey();
                return;
            }
            var selectedSlotData = availableSlotsMap[choiceNum];
            DateTime date = selectedSlotData.date;
            string time = selectedSlotData.time;

            var app = new Appointment(fullname, purpose, date, time, false, "APPOINTMENT");
            MyAppointments.Add(app);
            Program.Appointments.Add(app);
            Program.SaveAppointmentsToFile();
            Console.WriteLine($"\nAppointment booked successfully on {date:yyyy-MM-dd} at {time}!");
            Console.WriteLine("Press any key to return...");
            Console.ReadKey();
        }
        public static void AssignWalkInSlot(Receptionist staff)
        {
            Console.Write("\nEnter walk-in patient's Full Name: ");
            string walkinName = Console.ReadLine();
            Console.Write("Enter purpose: ");
            string purpose = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(purpose))
            {
                Console.WriteLine("Purpose of visit cannot be empty. Walk-in assignment cancelled.");
                Console.ReadKey();
                return;
            }

            Patient patient = Program.GetPatientByName(walkinName);
            if (patient == null)
            {
                Console.WriteLine("\n--- NEW WALK-IN PATIENT REGISTRATION ---");

                string u = $"WALKIN_{Guid.NewGuid().ToString().Substring(0, 4)}";
                string pw = "walkin_default";

                Console.Write("Enter Age: ");
                int age; int.TryParse(Console.ReadLine(), out age);
                Console.Write("Enter contact number: ");
                string contact = Console.ReadLine();
                Console.Write("Enter Address: ");
                string address = Console.ReadLine();
                Console.Write("Contact Person in case of emergency: ");
                string contactPerson = Console.ReadLine();
                Console.Write("Contact number of the contact person: ");
                string contactNo = Console.ReadLine();

                patient = new Patient(u, pw, walkinName, age, contact, address, contactPerson, contactNo);
                Program.Patients.Add(patient);
                Program.SavePatients();
                Console.WriteLine("New patient registered successfully!");

            }
            else
            {
                Console.WriteLine($"\nPatient '{walkinName}' found (Age: {patient.Age}).");
            }

            DateTime today = DateTime.Today;
            var appointments = Program.Appointments;

            Console.Clear();
            Console.WriteLine("=== Weekly Appointment Calendar ===");
            Console.Write("Time".PadRight(8));
            for (int d = 0; d < 7; d++)
            {
                Console.Write(today.AddDays(d).ToString("MM-dd").PadRight(8));
            }
            Console.WriteLine();

            int slotIndex = 1;
            var availableSlotsMap = new Dictionary<int, (DateTime date, string time)>();
            foreach (string slot in Program.TimeSlots)
            {
                Console.Write(slot.PadRight(8));

                for (int d = 0; d < 7; d++)
                {
                    DateTime day = today.AddDays(d);

                    bool booked = appointments.Any(a =>
                    {
                        if (a.Type == "FREE" || a.Type == "BUSY") return false;
                        if (a.Date.Date != day.Date) return false;
                        try
                        {
                            TimeSpan apptTime = TimeSpan.Parse(a.Time.Trim());
                            TimeSpan slotTime = TimeSpan.Parse(slot.Trim());
                            return apptTime == slotTime;
                        }
                        catch
                        {
                            return false;
                        }
                    });

                    Console.ForegroundColor = booked ? ConsoleColor.Red : ConsoleColor.Green;

                    if (booked)
                    {
                        Console.Write("[X]".PadRight(8));
                    }
                    else
                    {
                        Console.Write($"[{slotIndex}]".PadRight(8));
                        availableSlotsMap.Add(slotIndex, (day, slot));
                        slotIndex++;
                    }

                    Console.ResetColor();
                }
                Console.WriteLine();
            }

            Console.WriteLine("\nLegend: [N] Available | [X] Booked");

            if (availableSlotsMap.Count == 0)
            {
                Console.WriteLine("\nNo available slots for the next week.");
                Console.WriteLine("Press any key to return...");
                Console.ReadKey();
                return;
            }

            Console.Write("\nEnter the number of the slot you want to assign: ");
            if (!int.TryParse(Console.ReadLine(), out int choiceNum) || !availableSlotsMap.ContainsKey(choiceNum))
            {
                Console.WriteLine("Invalid selection or slot is not available. Assignment cancelled.");
                Console.ReadKey();
                return;
            }

            var selectedSlotData = availableSlotsMap[choiceNum];
            DateTime date = selectedSlotData.date;
            string time = selectedSlotData.time;
            var newAppointment = new Appointment(patient.fullname, purpose, date, time, true, "WALKIN");
            Program.Appointments.Add(newAppointment);

            if (!Program.Queue.WaitingList.Any(p => p.fullname.Equals(patient.fullname, StringComparison.OrdinalIgnoreCase)))
            {
                Program.Queue.WaitingList.Add(patient);
                Program.Queue.SortByPriority();
            }

            Program.SaveAppointmentsToFile();
            Console.WriteLine($"Walk-in for {patient.fullname} assigned successfully at {date.ToShortDateString()} {time}!");
            Console.ReadKey();
        }

        public void ViewAppointment()
        {
            Console.WriteLine("\nYOUR APPOINTMENTS");

            if (MyAppointments.Count == 0)
                Console.WriteLine("No appointments found.");
            else
            {
                var table = new ConsoleTable("No.", "Patient", "Purpose", "Date", "Time", "Type");
                for (int i = 0; i < MyAppointments.Count; i++)
                {
                    var a = MyAppointments[i];
                    table.AddRow(i + 1, a.PatientName, a.Purpose, a.Date.ToString("MMM dd, yyyy"), a.Time, a.IsWalkIn ? "Walk-in" : "Appointment");
                }
                table.Write();
            }
        }
        public void CancelAppointment()
        {
            Console.Write("\nEnter your purpose to cancel appointment: ");
            string purpose = Console.ReadLine();
            var app = MyAppointments.FirstOrDefault(a => a.Purpose.Equals(purpose, StringComparison.OrdinalIgnoreCase));
            if (app != null)
            {
                MyAppointments.Remove(app);
                Program.Appointments.Remove(app);
                Program.SaveAppointmentsToFile();
                Console.WriteLine("Appointment cancelled.");
            }
            else
            {
                Console.WriteLine("No appointment found with that purpose.");
            }
        }
    }

    public class Doctor : User
    {
        private string DoctorID { get; set; }

        public Doctor(string username, string password, string fullName, string doctorID)
            : base(username, password, fullName)
        {
            DoctorID = doctorID;
        }

        public override void DisplayMenu()
        {
            bool running = true;
            while (running)
            {
                Console.WriteLine($"\n---------- DOCTOR MENU ----------");
                Console.WriteLine("[1] View Queue");
                Console.WriteLine("[2] View Appointments");
                Console.WriteLine("[3] Cancel Appointment");
                Console.WriteLine("[4] Prescribe Medicine");
                Console.WriteLine("[5] View Patient Medical History");
                Console.WriteLine("[6] Add Patient Medical History Record");
                Console.WriteLine("[7] Logout");
                Console.Write("Enter choice: ");
                if (!int.TryParse(Console.ReadLine(), out int choice)) choice = -1;

                switch (choice)
                {
                    case 1:
                        Program.ViewCombinedServiceOrderForDoctor();
                        break;
                    case 2:
                        ViewAppointments();
                        break;
                    case 3:
                        CancelAppointment();
                        break;
                    case 4:
                        Prescribe();
                        break;
                    case 5:
                        ViewPatientHistory();
                        break;
                    case 6:
                        AddPatientHistoryRecord();
                        break;
                    case 7:
                        running = false;
                        Logout();
                        break;
                    default:
                        Console.WriteLine("Invalid choice. Try again.");
                        break;
                }
            }
        }
        public void ViewAppointments()
        {
            Console.WriteLine("\nAPPOINTMENTS (APPOINTMENT type only, sorted by date/time)");
            var apptsOnly = Program.Appointments
                .Where(a => a.Type == "APPOINTMENT")
                .OrderBy(a => a.Date).ThenBy(a => TimeSpan.Parse(a.Time))
                .ToList();

            if (apptsOnly.Count == 0)
            {
                Console.WriteLine("No appointments available.");
                return;
            }

            var table = new ConsoleTable("No.", "Patient", "Purpose", "Date", "Time");
            for (int i = 0; i < apptsOnly.Count; i++)
            {
                var a = apptsOnly[i];
                table.AddRow(i + 1, a.PatientName, a.Purpose, a.Date.ToString("MMM dd, yyyy"), a.Time);
            }
            table.Write();
        }
        public void CancelAppointment()
        {
            Console.Write("\nEnter patient's name to cancel appointment: ");
            string name = Console.ReadLine();
            Console.Write("Enter purpose: ");
            string purpose = Console.ReadLine();

            Appointment app = null;
            if (string.IsNullOrWhiteSpace(purpose))
            {
                app = Program.Appointments.FirstOrDefault(a => a.Type == "APPOINTMENT" && a.PatientName.Equals(name, StringComparison.OrdinalIgnoreCase));
            }
            else
            {
                app = Program.Appointments.FirstOrDefault(a => a.Type == "APPOINTMENT" && a.PatientName.Equals(name, StringComparison.OrdinalIgnoreCase) && a.Purpose.Equals(purpose, StringComparison.OrdinalIgnoreCase));
            }

            if (app != null)
            {
                Program.Appointments.Remove(app);
                var patient = Program.GetPatientByName(name);
                if (patient != null)
                {
                    var papp = patient.MyAppointments.FirstOrDefault(a => a.Purpose == app.Purpose && a.Date == app.Date && a.Time == app.Time && !a.IsWalkIn);
                    if (papp != null) patient.MyAppointments.Remove(papp);
                }
                Program.SaveAppointmentsToFile();
                Console.WriteLine($"Appointment for {name} canceled successfully.");
            }
            else Console.WriteLine("No matching appointment found.");
        }
        public void Prescribe()
        {
            var patient = Program.SearchAndSelectPatient("Enter Patient's Name to Prescribe for");
            if (patient == null)
            {
                Console.WriteLine("Operation cancelled.");
                Console.ReadKey();
                return;
            }

            Console.Write("Enter Prescription Details: ");
            string prescription = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(prescription))
            {
                Console.WriteLine("No prescription entered. Operation cancelled.");
                return;
            }

            string record = $"{DateTime.Now:yyyy-MM-dd} - Prescription by Dr. {fullname}: {prescription}";
            MedicalHistoryManager.AddRecord(patient.fullname, record);
            Console.WriteLine($"Prescription recorded successfully for {patient.fullname}.");
            Console.ReadKey();
        }

        public void ViewPatientHistory()
        {
            var patient = Program.SearchAndSelectPatient("Enter Patient's Name to View History for");
            if (patient == null)
            {
                Console.WriteLine("Operation cancelled.");
                Console.ReadKey();
                return;
            }

            MedicalHistoryManager.ViewHistory(patient.fullname);
        }

        public void AddPatientHistoryRecord()
        {
            var patient = Program.SearchAndSelectPatient("Enter Patient's Name to Add Record for");
            if (patient == null)
            {
                Console.WriteLine("Operation cancelled.");
                return;
            }

            Console.WriteLine($"\n--- Adding Record for {patient.fullname} ---");

            MedicalHistoryManager.AddIntakeRecordFromDoctor(patient);

            Console.Write("\nEnter additional Diagnosis:");
            string record = Console.ReadLine();

            if (!string.IsNullOrWhiteSpace(record))
            {
                string fullRecord = $"{DateTime.Now:yyyy-MM-dd} - Doctor {fullname} Diagnosis: {record}";
                MedicalHistoryManager.AddRecord(patient.fullname, fullRecord);
            }
            else
            {
                Console.WriteLine("Additional diagnosis skipped.");
            }

            Console.WriteLine("\nMedical history record(s) recorded successfully!");
           
        }
    }
    public class Appointment : Imanage
    {
        public string PatientName { get; set; }
        public string Purpose { get; set; }
        public DateTime Date { get; set; }
        public string Time { get; set; }
        public bool IsWalkIn { get; set; } = false;
        public string Type { get; set; } = "APPOINTMENT";

        public Appointment(string patient, string purpose, DateTime date, string time, bool isWalkIn = false, string type = "APPOINTMENT")
        {
            PatientName = patient;
            Purpose = purpose;
            Date = date;
            Time = time;
            IsWalkIn = isWalkIn;
            Type = type;
        }

        public void Add() { Console.WriteLine("Appointment added successfully."); }
        public void Update() { Console.WriteLine("Appointment updated successfully."); }
        public void Delete() { Console.WriteLine("Appointment canceled."); }
        public void View() { Console.WriteLine($"Patient: {PatientName} | Purpose: {Purpose} | Date: {Date:MMM dd, yyyy} | Time: {Time} | Type: {Type}"); }
    }

    public class QueueManager : Imanage
    {
        public List<Patient> WaitingList = new List<Patient>();
        public List<Patient> ServedPatients = new List<Patient>();

        public void AddPatientToQueue(Patient patient)
        {
            if (WaitingList.Any(p => p.fullname.Equals(patient.fullname, StringComparison.OrdinalIgnoreCase)))
            {
                Console.WriteLine("Patient is already in the walk-in queue.");
                return;
            }

            WaitingList.Add(patient);
            SortByPriority();
            Add();
            Program.SaveAppointmentsToFile();
        }

        public void SortByPriority()
        {
            WaitingList = WaitingList
                .OrderBy(p => Program.GetQueueArrivalIndex(p))
                .ToList();
        }
        public void ServeNextPatient()
        {
            var nextAppointment = Program.Appointments
                .Where(a => a.Type == "APPOINTMENT")
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .FirstOrDefault();

            var nextWalkInAppointment = Program.Appointments
                .Where(a => a.Type == "WALKIN")
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .FirstOrDefault();

            dynamic patientToServe = null;
            Appointment appointmentToRemove = null;
            string type = "";

            DateTime? apptDateTime = nextAppointment != null ? nextAppointment.Date.Date.Add(TimeSpan.Parse(nextAppointment.Time)) : (DateTime?)null;
            DateTime? walkInDateTime = nextWalkInAppointment != null ? nextWalkInAppointment.Date.Date.Add(TimeSpan.Parse(nextWalkInAppointment.Time)) : (DateTime?)null;

            Patient originalWalkInPatient = null;
            if (nextWalkInAppointment != null)
            {
                originalWalkInPatient = WaitingList.FirstOrDefault(p => p.fullname.Equals(nextWalkInAppointment.PatientName, StringComparison.OrdinalIgnoreCase));
            }

            if (apptDateTime.HasValue && (!walkInDateTime.HasValue || apptDateTime.Value <= walkInDateTime.Value))
            {
                patientToServe = Program.GetPatientByName(nextAppointment.PatientName);
                appointmentToRemove = nextAppointment;
                type = "Appointment";
            }
            else if (walkInDateTime.HasValue)
            {
                patientToServe = originalWalkInPatient;
                appointmentToRemove = nextWalkInAppointment;
                type = "Walk-in";
            }
            else
            {
                Console.WriteLine("No patients to serve.");
                return;
            }

            Console.WriteLine($"Serving ({type}) {patientToServe.fullname}...");
            ServedPatients.Add(patientToServe);

            if (type == "Walk-in")
            {
                var walkinPatient = WaitingList.FirstOrDefault(p => p.fullname.Equals(patientToServe.fullname, StringComparison.OrdinalIgnoreCase));
                if (walkinPatient != null)
                {
                    WaitingList.Remove(walkinPatient);
                }
            }

            if (appointmentToRemove != null)
            {
                Program.Appointments.Remove(appointmentToRemove);

                var p = Program.GetPatientByName(appointmentToRemove.PatientName);
                if (p != null)
                {
                    var papp = p.MyAppointments.FirstOrDefault(a => a.Purpose == appointmentToRemove.Purpose && a.Date == appointmentToRemove.Date && a.Time == appointmentToRemove.Time && a.Type == appointmentToRemove.Type);
                    if (papp != null) p.MyAppointments.Remove(papp);
                }
            }

            Program.SaveAppointmentsToFile();
        }
        public void SkipPatient()
        {
            var nextAppointment = Program.Appointments
                .Where(a => a.Type == "APPOINTMENT")
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .FirstOrDefault();

            var nextWalkInAppointment = Program.Appointments
                .Where(a => a.Type == "WALKIN")
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .FirstOrDefault();

            Appointment targetAppointment = null;
            DateTime? apptDateTime = nextAppointment != null ? nextAppointment.Date.Date.Add(TimeSpan.Parse(nextAppointment.Time)) : (DateTime?)null;
            DateTime? walkInDateTime = nextWalkInAppointment != null ? nextWalkInAppointment.Date.Date.Add(TimeSpan.Parse(nextWalkInAppointment.Time)) : (DateTime?)null;

            if (!apptDateTime.HasValue && !walkInDateTime.HasValue)
            {
                Console.WriteLine("No patients available in the queue to skip.");
                return;
            }

            if (apptDateTime.HasValue && (!walkInDateTime.HasValue || apptDateTime.Value <= walkInDateTime.Value))
            {
                targetAppointment = nextAppointment;
            }
            else if (walkInDateTime.HasValue)
            {
                targetAppointment = nextWalkInAppointment;
            }

            if (targetAppointment.Type == "APPOINTMENT")
            {
                Console.WriteLine($"Cannot skip scheduled appointment for {targetAppointment.PatientName}. Please cancel or reschedule formally.");
                return;
            }
            else if (targetAppointment.Type == "WALKIN")
            {
                Patient patientToMove = Program.GetPatientByName(targetAppointment.PatientName);

                if (WaitingList.Count < 2)
                {
                    Console.WriteLine("Cannot skip walk-in. Only one waiting.");
                    return;
                }

                if (patientToMove != null)
                {
                    WaitingList.Remove(patientToMove);
                    WaitingList.Add(patientToMove);

                    Program.Queue.SortByPriority();

                    Console.WriteLine($"{patientToMove.fullname} (Walk-in at {targetAppointment.Time}) has been skipped temporarily.");
                    Program.SaveAppointmentsToFile();
                }
                else
                {
                    Console.WriteLine($"Error: Walk-in patient {targetAppointment.PatientName} not found in waiting list.");
                }
            }
        }

        public void Add() { Console.WriteLine("Patient added to walk-in queue successfully!"); }
        public void Update() { Console.WriteLine("Queue updated successfully!"); }
        public void Delete() { Console.WriteLine("Successfully deleted!"); }

        public void View()
        {
            Console.WriteLine("\nCURRENT WALK-IN QUEUE");
            if (WaitingList.Count == 0) { Console.WriteLine("No walk-in patients waiting.\n"); return; }

            var table = new ConsoleTable("No.", "Full Name", "Contact", "Contact Person");
            for (int i = 0; i < WaitingList.Count; i++)
            {
                Patient p = WaitingList[i];
                table.AddRow(i + 1, p.fullname, p.Contact, p.ContactPerson);
            }
            table.Write();
        }
    }

    public static class MedicalHistoryManager
    {
        public static Dictionary<string, List<string>> Records = new Dictionary<string, List<string>>(StringComparer.OrdinalIgnoreCase);

        public static void AddRecord(string patientName, string record)
        {
            if (!Records.ContainsKey(patientName))
            {
                Records.Add(patientName, new List<string>());
            }
            Records[patientName].Add(record);
            Program.SavePatients();
        }
        public static void AddIntakeRecordFromDoctor(Patient patient)
        {
            Console.WriteLine($"\n---------- MEDICAL HISTORY INTAKE FORM for {patient.fullname} ----------\n");

            var options = new List<string>
            {
                "Allergies", "Anemia", "Anxiety", "Arthritis", "Asthma", "Blood transfusion", "Cancer",
                "Congestive heart failure", "COPD / Emphysema", "Depression", "Diabetes (Type 1 / Type 2)",
                "Epilepsy / Seizures", "GERD (Acid Reflux)", "Glaucoma", "Heart Disease", "Hypertension",
                "High Cholesterol", "Kidney Disease", "Liver Disease", "Migraines", "Osteoporosis",
                "Stroke", "Ulcers", "No known medical conditions", "Other (Specify)"
            };

            var selected = AnsiConsole.Prompt(
                new MultiSelectionPrompt<string>()
                    .Title("[yellow]Select all applicable medical conditions:[/]")
                    .Required(false)
                    .PageSize(20)
                    .MoreChoicesText("[grey](Scroll to see more)[/]")
                    .InstructionsText("[grey](Use ↑/↓ to navigate, space to select, enter to confirm)[/]")
                    .AddChoices(options)
            );

            if (selected.Contains("Other (Specify)"))
            {
                Console.Write("Please specify the 'Other' condition: ");
                string other = Console.ReadLine();
                selected.Remove("Other (Specify)");
                if (!string.IsNullOrWhiteSpace(other)) selected.Add(other);
            }

            foreach (var condition in selected)
            {
                if (condition == "No known medical conditions" && selected.Count > 1) continue;
                if (string.IsNullOrWhiteSpace(condition)) continue;
                string record = $"{DateTime.Now:yyyy-MM-dd} - Intake Condition: {condition}";

                if (!Records.ContainsKey(patient.fullname) || !Records[patient.fullname].Contains(record))
                {
                    AddRecord(patient.fullname, record);
                }
            }

            Console.WriteLine("\nMedical history record(s) recorded successfully by Doctor.\n");
        }
        public static void ViewHistory(string patientName)
        {
            Console.WriteLine($"\nMEDICAL HISTORY of {patientName}");
            if (!Records.ContainsKey(patientName) || Records[patientName].Count == 0)
            {
                Console.WriteLine("No medical history available.");
            }
            else
            {
                foreach (var record in Records[patientName])
                {
                    Console.WriteLine($"- {record}");
                }
            }
            Console.ReadKey();
        }

        public static string SerializeHistory(string patientName)
        {
            if (Records.ContainsKey(patientName))
            {
                return string.Join(";", Records[patientName]);
            }
            return string.Empty;
        }

        public static void DeserializeHistory(string patientName, string historyString)
        {
            if (!string.IsNullOrWhiteSpace(historyString))
            {
                var historyRecords = historyString.Split(new[] { ';' }, StringSplitOptions.RemoveEmptyEntries).Select(s => s.Trim()).ToList();
                Records[patientName] = historyRecords;
            }
            else if (!Records.ContainsKey(patientName))
            {
                Records[patientName] = new List<string>();
            }
        }
    }
    public class Program
    {
        public static QueueManager Queue = new QueueManager();
        public static List<Appointment> Appointments = new List<Appointment>();
        public static List<Patient> Patients = new List<Patient>();
        public static Doctor AttendingDoctor;
        public static string PatientFile = "patients.txt";
        public static string AppointmentFile = "appointments.txt";
        public static string ArchiveFolder = "Archive";
        public static readonly string[] TimeSlots = { "08:30", "09:00", "09:30", "10:00", "10:30", "11:00", "11:30", "13:00", "13:30", "14:00", "14:30", "15:00", "15:30", "16:00", "16:30", "17:00", "17:30", "18:00", "18:30" };
        public static Patient SearchAndSelectPatient(string prompt)
        {
            var allPatients = Program.Patients
                .Select(p => new
                {
                    Patient = p,
                    Display = $"{p.fullname} (Username: {p.username}, Contact: {p.Contactno})"
                })
                .ToList();

            if (allPatients.Count == 0)
            {
                AnsiConsole.MarkupLine("[red]No patient records available for search.[/]");
                Console.ReadKey();
                return null;
            }

            Console.Clear();
            var searchPrompt = AnsiConsole.Prompt(
                new TextPrompt<string>($"\n[bold yellow]{prompt}[/]:")
                .AllowEmpty()
            ).ToLower();

            if (string.IsNullOrWhiteSpace(searchPrompt))
            {
                AnsiConsole.MarkupLine("[red]Search cancelled.[/]");
                Console.ReadKey();
                return null;
            }

            var filteredMatches = allPatients
                .Where(p =>
                    p.Patient.fullname.ToLower().Contains(searchPrompt) ||
                    p.Patient.username.ToLower().Contains(searchPrompt))
                .ToList();

            if (filteredMatches.Count == 0)
            {
                AnsiConsole.MarkupLine($"[red]No patients found matching '{searchPrompt}'.[/]");
                Console.ReadKey();
                return null;
            }

            var selectedItem = AnsiConsole.Prompt(
                new SelectionPrompt<string>()
                    .Title($"[green]Found {filteredMatches.Count} match(es). Select the patient:[/]")
                    .PageSize(10)
                    .AddChoices(filteredMatches.Select(m => m.Display))
            );

            var finalPatient = filteredMatches.FirstOrDefault(m => m.Display == selectedItem);

            if (finalPatient != null)
            {
                AnsiConsole.MarkupLine($"[green]Selected Patient: {finalPatient.Patient.fullname}[/]");
                return finalPatient.Patient;
            }

            return null;
        }

        static void Main(string[] args)
        {
            Console.Title = "Clinic Queue Management System";

            if (!Directory.Exists(ArchiveFolder)) Directory.CreateDirectory(ArchiveFolder);

            LoadPatients();
            LoadAppointments();

            Receptionist staff = new Receptionist("staff1", "1234", "Nurse Veronica", "S001");
            AttendingDoctor = new Doctor("doctor1", "5678", "Dr. Uno", "D001");

            bool running = true;

            while (running)
            {
                AnsiConsole.Clear();
                var headerPanel = new Panel("[bold blue]CLINIC QUEUE MANAGEMENT SYSTEM[/]")
                    .Border(BoxBorder.Double)
                    .BorderColor(Color.Yellow)
                    .Padding(1, 1, 1, 1);

                AnsiConsole.Write(Align.Center(headerPanel));
                Console.WriteLine();

                var selectionPrompt = new SelectionPrompt<string>()
                    .Title("[yellow]Select an option:[/]")
                    .AddChoices(new[] { "Staff Login", "Patient Login / Register", "Doctor Login", "Exit" });

                var choice = AnsiConsole.Prompt(selectionPrompt);
                string processedChoice = choice;

                switch (processedChoice)
                {
                    case "Staff Login":
                        Console.Write("Enter Username: ");
                        string suser = Console.ReadLine();
                        Console.Write("Enter Password: ");
                        string spwd = Console.ReadLine();
                        if (suser == staff.username && spwd == staff.password) staff.DisplayMenu();
                        else Console.WriteLine("Invalid staff credentials!");
                        break;

                    case "Patient Login / Register":
                        Console.WriteLine("[1] Login as Existing Patient");
                        Console.WriteLine("[2] Register as New Patient ");
                        Console.Write("\nEnter choice: ");
                        string option = Console.ReadLine();

                        if (option == "1")
                        {
                            Console.Write("\nEnter Username: ");
                            string uname = Console.ReadLine().Trim();
                            Console.Write("Enter Password: ");
                            string pwd = Console.ReadLine().Trim();

                            var existingPatient = Patients.FirstOrDefault(p => p.username.Equals(uname, StringComparison.OrdinalIgnoreCase) && p.password == pwd);

                            if (existingPatient != null)
                            {
                                Console.WriteLine($"Welcome back, {existingPatient.fullname}!");
                                existingPatient.DisplayMenu();
                            }
                            else
                            {
                                Console.WriteLine("Invalid credentials! Please register.");
                            }
                        }
                        else if (option == "2")
                        {
                            Console.Write("\nEnter Full Name: ");
                            string name = Console.ReadLine();
                            var existing = Patients.FirstOrDefault(p => p.fullname.Equals(name, StringComparison.OrdinalIgnoreCase));
                            if (existing != null) { Console.WriteLine("An account with this name already exists. Please log in instead."); break; }

                            Console.Write("Set Username: ");
                            string u = Console.ReadLine();
                            Console.Write("Set Password: ");
                            string pw = Console.ReadLine();
                            Console.Write("Enter Age: ");
                            int age; int.TryParse(Console.ReadLine(), out age);
                            Console.Write("Enter Contact number: ");
                            string contact = Console.ReadLine();
                            Console.Write("Enter Address: ");
                            string address = Console.ReadLine();

                            Console.Write("Contact Person in case of emergency: ");
                            string contactPerson = Console.ReadLine();
                            Console.Write("Contact number of the Contact Person: ");
                            string contactNo = Console.ReadLine();

                            var newPatient = new Patient(u, pw, name, age, contact, address, contactPerson, contactNo);
                            Patients.Add(newPatient);
                            SavePatients();
                            Console.WriteLine("Registered successfully!");

                            // No automatic medical history collection during registration
                        }
                        else Console.WriteLine("Invalid choice.");
                        break;

                    case "Doctor Login":
                        Console.Write("\nDoctor Username: ");
                        string duser = Console.ReadLine();
                        Console.Write("Doctor Password: ");
                        string dpwd = Console.ReadLine();
                        if (duser == AttendingDoctor.username && dpwd == AttendingDoctor.password) AttendingDoctor.DisplayMenu();
                        else Console.WriteLine("Invalid doctor credentials!");
                        break;

                    case "Exit":
                        running = false;
                        break;

                    default:
                        Console.WriteLine("Invalid choice. Try again.");
                        break;
                }

                if (running)
                {
                    Console.Write("\nDo you want to continue? (Y/N): ");
                    string again = Console.ReadLine().ToUpper();
                    if (again == "N") running = false;
                }
            }
            SavePatients();
            SaveAppointmentsToFile();
        }

        public static void SavePatients()
        {
            using (StreamWriter writer = new StreamWriter(PatientFile))
            {
                foreach (var p in Patients)
                {
                    string history = MedicalHistoryManager.SerializeHistory(p.fullname);
                    writer.WriteLine($"{p.username}|{p.password}|{p.fullname}|{p.Age}|{p.Contact}|{p.Address}|{p.ContactPerson}|{p.Contactno}|{history}");
                }
            }
        }
        public static void LoadPatients()
        {
            try
            {
                Patients.Clear();
                MedicalHistoryManager.Records.Clear();

                if (!File.Exists(PatientFile))
                {
                    Console.WriteLine("Patient file not found. No patient records loaded.");
                    return;
                }

                foreach (var line in File.ReadAllLines(PatientFile))
                {
                    if (string.IsNullOrWhiteSpace(line))
                        continue;

                    var data = line.Split('|');

                    if (data.Length < 9)
                    {
                        Console.WriteLine($"Skipping corrupted or incomplete record: {line}");
                        continue;
                    }

                    for (int i = 0; i < data.Length; i++)
                        data[i] = data[i].Trim();

                    int age = 0;
                    if (!int.TryParse(data[3], out age))
                    {
                        Console.WriteLine($"Invalid age format for patient: {data[2]}. Defaulting age to 0.");
                    }
                    Patient newPatient = new Patient(
                        data[0], // username
                        data[1], // password
                        data[2], // fullname
                        age,
                        data[4], // contact
                        data[5], // address
                        data[6], // contactPerson
                        data[7]  // contactNo
                    );

                    MedicalHistoryManager.DeserializeHistory(newPatient.fullname, data[8]);

                    Patients.Add(newPatient);
                }

                Console.WriteLine("Patient records successfully loaded.");
            }
            catch (IOException ioEx)
            {
                Console.WriteLine($"File I/O error while loading patient data! Error: {ioEx.Message}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Unexpected error occurred while loading patients! Error: {ex.Message}");
            }
        }
        public static void SaveAppointmentsToFile()
        {
            using (StreamWriter writer = new StreamWriter(AppointmentFile))
            {
                foreach (var a in Appointments)
                {
                    if (a.Type == "BUSY")
                        writer.WriteLine($"BUSY | {a.Purpose} | {a.Date:yyyy-MM-dd} | {a.Time}");
                    else if (a.Type == "WALKIN")
                        writer.WriteLine($"WALKIN | {a.PatientName} | {a.Purpose} | {a.Date:yyyy-MM-dd} | {a.Time}");
                    else
                        writer.WriteLine($"APPOINTMENT | {a.PatientName} | {a.Purpose} | {a.Date:yyyy-MM-dd} | {a.Time}");
                }
            }
        }

        public static void LoadAppointments()
        {
            Appointments.Clear();
            Queue.WaitingList.Clear();
            if (!File.Exists(AppointmentFile)) File.WriteAllText(AppointmentFile, "");
            var lines = File.ReadAllLines(AppointmentFile);
            foreach (var line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var parts = line.Split('|').Select(p => p.Trim()).ToArray();
                var key = parts[0].ToUpper();
                if (key == "BUSY" && parts.Length >= 4)
                {
                    if (DateTime.TryParse(parts[2], out DateTime d)) Appointments.Add(new Appointment("", parts[1], d, parts[3], false, "BUSY"));
                }
                else if (key == "WALKIN" && parts.Length >= 5)
                {
                    if (DateTime.TryParse(parts[3], out DateTime d))
                    {
                        var a = new Appointment(parts[1], parts[2], d, parts[4], true, "WALKIN");
                        Appointments.Add(a);
                        var patient = GetPatientByName(parts[1]);

                        if (patient == null) patient = new Patient(parts[1], "", parts[1], 0, "", "", "", "");
                        Queue.WaitingList.Add(patient);
                    }
                }
                else if (key == "APPOINTMENT" && parts.Length >= 5)
                {
                    if (DateTime.TryParse(parts[3], out DateTime d))
                    {
                        var a = new Appointment(parts[1], parts[2], d, parts[4], false, "APPOINTMENT");
                        Appointments.Add(a);
                        var patient = GetPatientByName(parts[1]);
                        if (patient != null) patient.MyAppointments.Add(a);
                    }
                }
            }
            Queue.SortByPriority();
        }

        public static List<string> GetFreeSlotsForDate(DateTime date)
        {
            var taken = new HashSet<string>();
            foreach (var a in Appointments)
            {
                if (a.Date.Date == date.Date)
                    taken.Add(a.Time);
            }
            return TimeSlots.Where(s => !taken.Contains(s)).ToList();
        }

        public static void ManageCalendar(Receptionist receptionist)
        {
            while (true)
            {
                Console.Clear();

                Console.WriteLine("[1] Free a slot");
                Console.WriteLine("[2] Assign walk-in to free slot (with registration)");
                Console.WriteLine("[3] Back");
                Console.Write("Choice: ");

                string choice = Console.ReadLine().Trim();

                if (choice == "1")
                {
                    Console.Write("Enter date to modify (YYYY-MM-DD): ");
                    if (!DateTime.TryParse(Console.ReadLine(), out DateTime selectedDate))
                    {
                        Console.WriteLine("Invalid date.");
                        Console.ReadKey();
                        continue;
                    }

                    Console.WriteLine("Available time slots:");
                    for (int i = 0; i < TimeSlots.Length; i++)
                    {
                        Console.WriteLine($"{i + 1}. {TimeSlots[i]}");
                    }

                    Console.Write("Select slot number to free: ");
                    if (!int.TryParse(Console.ReadLine(), out int slotNum) || slotNum < 1 || slotNum > TimeSlots.Length)
                    {
                        Console.WriteLine("Invalid slot.");
                        Console.ReadKey();
                        continue;
                    }

                    string selectedSlot = TimeSlots[slotNum - 1];

                    var appointments = File.ReadAllLines(AppointmentFile).ToList();

                    int removedCount = appointments.RemoveAll(a =>
                    {
                        var p = a.Split('|').Select(x => x.Trim()).ToArray();
                        if (p.Length < 4) return false;

                        string entryDate = p[0] == "BUSY" ? p[2] : p[3];
                        string entryTime = p[0] == "BUSY" ? p[3] : p[4];
                        return entryDate == selectedDate.ToString("yyyy-MM-dd") && entryTime == selectedSlot;
                    });

                    if (removedCount > 0)
                    {
                        File.WriteAllLines(AppointmentFile, appointments);
                        LoadAppointments();
                        Console.WriteLine($"Successfully freed {removedCount} slot(s) at {selectedDate:yyyy-MM-dd} {selectedSlot}.");
                    }
                    else
                    {
                        Console.WriteLine("No block or appointment found for that slot.");
                    }
                    Console.WriteLine("Press any key to continue...");
                    Console.ReadKey();
                }
                else if (choice == "2")
                {
                    Patient.AssignWalkInSlot(receptionist);
                }
                else if (choice == "3")
                {
                    return;
                }
                else
                {
                    Console.WriteLine("Invalid choice. Press any key to try again...");
                    Console.ReadKey();
                }
            }
        }
        public static void DisplayCalendarForPatient()
        {
            DateTime today = DateTime.Today;
            if (!File.Exists(AppointmentFile)) File.WriteAllText(AppointmentFile, "");
            var appointments = File.ReadAllLines(AppointmentFile);

            Console.Clear();
            Console.WriteLine("=== Weekly Calendar ===");
            Console.Write("Time".PadRight(8));
            for (int d = 0; d < 7; d++) Console.Write(today.AddDays(d).ToString("MM-dd").PadRight(12));
            Console.WriteLine();

            foreach (string slot in TimeSlots)
            {
                Console.Write(slot.PadRight(8));
                for (int d = 0; d < 7; d++)
                {
                    DateTime day = today.AddDays(d);
                    bool booked = appointments.Any(a =>
                    {
                        var p = a.Split('|');
                        return p[0] != "BUSY" && p.Length >= 5 && p[3] == day.ToString("yyyy-MM-dd") && p[4] == slot;
                    });

                    bool blocked = appointments.Any(a =>
                    {
                        var p = a.Split('|');
                        return p[0] == "BUSY" && p.Length >= 4 && p[2] == day.ToString("yyyy-MM-dd") && p[3] == slot;
                    });

                    Console.ForegroundColor = blocked ? ConsoleColor.Yellow : (booked ? ConsoleColor.Red : ConsoleColor.Green);
                    string mark = blocked ? "[B]" : (booked ? "[X]" : "[ ]");
                    Console.Write(mark.PadRight(12));
                    Console.ResetColor();
                }
                Console.WriteLine();
            }

            Console.WriteLine("\nLegend: [ ] Available | [X] Booked");
            Console.WriteLine("\nPress any key to return...");
            Console.ReadKey();
        }

        public static Patient GetPatientByName(string name)
        {
            var p = Patients.FirstOrDefault(x => x.fullname.Equals(name, StringComparison.OrdinalIgnoreCase));
            if (p != null) return p;
            p = Queue.WaitingList.Concat(Queue.ServedPatients).FirstOrDefault(x => x.fullname.Equals(name, StringComparison.OrdinalIgnoreCase));
            return p;
        }

        public static int GetQueueArrivalIndex(Patient p)
        {
            if (!File.Exists(AppointmentFile)) return int.MaxValue;
            var lines = File.ReadAllLines(AppointmentFile);
            for (int i = 0; i < lines.Length; i++)
            {
                var parts = lines[i].Split('|');
                if (parts[0].Trim().ToUpper() == "WALKIN" && parts.Length >= 2 && parts[1].Trim().Equals(p.fullname, StringComparison.OrdinalIgnoreCase))
                    return i;
            }
            return int.MaxValue;
        }
        public static void ViewAllAppointments()
        {
            Console.WriteLine("ALL SCHEDULED APPOINTMENTS ");

            var apptsOnly = Appointments
                .Where(a => a.Type == "APPOINTMENT")
                .OrderBy(a => a.Date).ThenBy(a => TimeSpan.Parse(a.Time)).ToList();

            if (apptsOnly.Count == 0) { Console.WriteLine("No scheduled appointments.\n"); return; }

            var table = new ConsoleTable("No.", "Patient", "Purpose", "Date", "Time");
            for (int i = 0; i < apptsOnly.Count; i++)
            {
                var a = apptsOnly[i];
                table.AddRow(i + 1, a.PatientName, a.Purpose, a.Date.ToString("MMM dd, yyyy"), a.Time);
            }
            table.Write();
        }

        public static void ViewCombinedServiceOrderForDoctor()
        {
            var today = DateTime.Today.Date;
            var allAppointments = Program.Appointments;
            var upcomingAppointments = allAppointments
                .Where(a => a.Type == "APPOINTMENT" && a.Date.Date == today)
                .OrderBy(a => a.Date)
                .ThenBy(a => TimeSpan.Parse(a.Time))
                .ToList();
            List<dynamic> walkinQueue = Program.Queue.WaitingList.Select(w =>
            {
                var walkinApp = allAppointments
                    .FirstOrDefault(a => a.Type == "WALKIN" && a.PatientName.Equals(w.fullname, StringComparison.OrdinalIgnoreCase) && a.Date.Date == today);

                return (dynamic)new
                {
                    Patient = w,
                    Appointment = walkinApp
                };
            })
                .Where(item => item.Appointment != null)
                .OrderBy(item => item.Appointment.Date)
                .ThenBy(item => TimeSpan.Parse(item.Appointment.Time))
                .ThenBy(item => Program.GetQueueArrivalIndex(item.Patient))
                .ToList();

            var combinedList = new List<dynamic>();

            foreach (var a in upcomingAppointments)
            {
                var p = Program.GetPatientByName(a.PatientName) ?? new Patient(a.PatientName, "", a.PatientName, 0, "", "", "", "");
                combinedList.Add(new
                {
                    Patient = p,
                    Appointment = a,
                    TypeDisplay = "Appointment",
                    ServiceTime = a.Date.Date.Add(TimeSpan.Parse(a.Time))
                });
            }

            foreach (var item in walkinQueue)
            {
                combinedList.Add(new
                {
                    Patient = item.Patient,
                    Appointment = item.Appointment,
                    TypeDisplay = "Walk-in",
                    ServiceTime = item.Appointment.Date.Date.Add(TimeSpan.Parse(item.Appointment.Time))
                });
            }

            var finalServiceOrder = combinedList
                .OrderBy(c => c.ServiceTime)
                .ToList();

            Console.WriteLine($"\n--- DOCTOR'S SERVICE ORDER QUEUE ({today:MMM dd, yyyy}) ---");
            if (finalServiceOrder.Count == 0)
            {
                Console.WriteLine("No patients in the queue or with assigned appointments for today.");
                return;
            }

            var tableCombined = new ConsoleTable("No.", "Patient Name", "Type", "Purpose", "Date", "Time");
            for (int i = 0; i < finalServiceOrder.Count; i++)
            {
                var c = finalServiceOrder[i];
                tableCombined.AddRow(
                    i + 1,
                    c.Patient.fullname,
                    c.TypeDisplay,
                    c.Appointment?.Purpose ?? "",
                    c.Appointment?.Date.ToString("MMM dd, yyyy") ?? "",
                    c.Appointment?.Time ?? ""
                );
            }
            tableCombined.Write();
            Console.WriteLine("\nPress any key to return...");
            Console.ReadKey();
        }
        public static void GenerateDailyReport()
        {
            Console.WriteLine("DAILY REPORT");
            Console.WriteLine($"Total Patients Served today: {Queue.ServedPatients.Count}");
        }

        public static void ArchiveAndResetAll()
        {
            string date = DateTime.Now.ToString("yyyy-MM-dd");

            if (File.Exists(PatientFile))
            {
                string patientsArchive = Path.Combine(ArchiveFolder, $"patients_{date}.txt");
                File.Copy(PatientFile, patientsArchive, true);
                Console.WriteLine($"Patients archived to {patientsArchive}");
            }

            if (File.Exists(AppointmentFile))
            {
                string appointmentsArchive = Path.Combine(ArchiveFolder, $"appointments_{date}.txt");
                File.Copy(AppointmentFile, appointmentsArchive, true);
                Console.WriteLine($"Appointments archived to {appointmentsArchive}");
                File.WriteAllText(AppointmentFile, string.Empty);
                Appointments.Clear();
                foreach (var patient in Patients) patient.MyAppointments.Clear();
            }

            string queueArchive = Path.Combine(ArchiveFolder, $"queue_{date}.txt");
            using (StreamWriter writer = new StreamWriter(queueArchive))
            {
                foreach (var p in Queue.WaitingList) writer.WriteLine($"{p.username},{p.password},{p.fullname},{p.Age},{p.Contact},{p.Address},{p.ContactPerson},{p.Contactno},Waiting");
                foreach (var p in Queue.ServedPatients) writer.WriteLine($"{p.username},{p.password},{p.fullname},{p.Age},{p.Contact},{p.Address},{p.ContactPerson},{p.Contactno},Served");
            }
            Queue.WaitingList.Clear();
            Queue.ServedPatients.Clear();
            Console.WriteLine($"Queue archived to {queueArchive}");
            Console.WriteLine("All appointments and queue archived and reset.");
        }
    }
}
