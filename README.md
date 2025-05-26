import React, { useState, useEffect, useRef, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, onSnapshot, addDoc, updateDoc, deleteDoc, doc, query, where, Timestamp, setDoc } from 'firebase/firestore';
import { useDrag, useDrop, DndProvider } from 'react-dnd';
import { HTML5Backend } from 'react-dnd-html5-backend';

// Define global variables for Firebase configuration and app ID
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-time-planner-app';
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Drag and Drop Item Types
const ItemTypes = {
  COURSE: 'course',
  SCHEDULED_COURSE: 'scheduled_course',
};

// Course definitions with colors and duration in 15-minute intervals
const coursesData = [
  { category: 'פסיכולוגיה', color: 'bg-sky-200', defaultText: 'פסיכולוגיה', durationIntervals: 4 }, // Default 1 hour
  { category: 'פוליטיקה וממשל', color: 'bg-yellow-200', defaultText: 'פוליטיקה וממשל', durationIntervals: 4 }, // Default 1 hour
  { category: 'כללי', color: 'bg-pink-300', defaultText: 'כללי', durationIntervals: 4 }, // Default 1 hour
  { category: 'הפסקה גדולה', color: 'bg-gray-300', defaultText: 'הפסקה גדולה', durationIntervals: 6 }, // 1.5 hours = 6 * 15min intervals
  { category: 'הפסקה רגילה', color: 'bg-gray-200', defaultText: 'הפסקה רגילה', durationIntervals: 4 }, // 1 hour = 4 * 15min intervals
];

// Constants for grid row heights (in px, based on 15-minute intervals)
const FIFTEEN_MIN_HEIGHT = 15; // Height for a 15-minute grid row

function App() {
  // State variables for Firebase instances and user ID
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  // State for navigation and selected date
  const [currentPage, setCurrentPage] = useState('monthly'); // 'monthly', 'todo', 'study-planner'
  const [selectedDate, setSelectedDate] = useState(new Date()); // Used for monthly calendar navigation & daily planner

  // State for tasks (events are now tasks)
  const [tasks, setTasks] = useState([]);
  // State for daily study schedule
  const [dailyStudySchedule, setDailyStudySchedule] = useState({}); // {date: {interval: {category, customText, durationIntervals}}}

  // State for modals
  const [showAddTaskModal, setShowAddTaskModal] = useState(false);
  const [showNotificationModal, setShowNotificationModal] = useState(false);
  const [notificationMessage, setNotificationMessage] = useState('');
  const [showAlertModal, setShowAlertModal] = useState(false); // For custom alert
  const [alertMessage, setAlertMessage] = useState(''); // For custom alert

  // Refs for modals to handle clicks outside
  const addTaskModalRef = useRef();
  const notificationModalRef = useRef();
  const alertModalRef = useRef(); // Ref for custom alert modal

  // Initialize Firebase and set up authentication listener
  useEffect(() => {
    try {
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const authentication = getAuth(app);
      setDb(firestore);
      setAuth(authentication);

      // Sign in with custom token or anonymously
      const signIn = async () => {
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(authentication, initialAuthToken);
          } else {
            await signInAnonymously(authentication);
          }
        } catch (error) {
          console.error("Firebase authentication error:", error);
        }
      };
      signIn();

      // Listen for authentication state changes
      const unsubscribeAuth = onAuthStateChanged(authentication, (user) => {
        if (user) {
          setUserId(user.uid);
          console.log("User ID:", user.uid); // Log the user ID for debugging
        } else {
          setUserId(null);
          console.log("No user signed in.");
        }
        setIsAuthReady(true); // Auth state is ready
      });

      return () => unsubscribeAuth(); // Cleanup auth listener on unmount
    } catch (error) {
      console.error("Failed to initialize Firebase:", error);
    }
  }, []);

  // Fetch tasks and daily study schedule when Firebase is ready and userId is available
  useEffect(() => {
    if (!db || !userId || !isAuthReady) return;

    // Fetch tasks
    const tasksCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/tasks`);
    const qTasks = query(tasksCollectionRef, where("userId", "==", userId)); // Filter by userId
    const unsubscribeTasks = onSnapshot(qTasks, (snapshot) => {
      const fetchedTasks = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        startDate: doc.data().startDate instanceof Timestamp ? doc.data().startDate.toDate() : new Date(doc.data().startDate),
        endDate: doc.data().endDate instanceof Timestamp ? doc.data().endDate.toDate() : new Date(doc.data().endDate),
        reminder: doc.data().reminder instanceof Timestamp ? doc.data().reminder.toDate() : (doc.data().reminder ? new Date(doc.data().reminder) : null)
      }));
      setTasks(fetchedTasks);
    }, (error) => {
      console.error("Error fetching tasks:", error);
    });

    // Fetch daily study schedules
    const studySchedulesCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/dailyStudySchedules`);
    const qStudySchedules = query(studySchedulesCollectionRef, where("userId", "==", userId));
    const unsubscribeStudySchedules = onSnapshot(qStudySchedules, (snapshot) => {
      const fetchedSchedules = {};
      snapshot.docs.forEach(doc => {
        // Ensure schedule object is deeply copied to avoid direct state mutation issues
        fetchedSchedules[doc.id] = JSON.parse(JSON.stringify(doc.data().schedule));
      });
      setDailyStudySchedule(fetchedSchedules);
    }, (error) => {
      console.error("Error fetching daily study schedules:", error);
    });


    return () => {
      unsubscribeTasks(); // Cleanup task listener
      unsubscribeStudySchedules(); // Cleanup study schedules listener
    };
  }, [db, userId, isAuthReady]);

  // Handle outside clicks for modals
  useEffect(() => {
    const handleClickOutside = (event) => {
      if (addTaskModalRef.current && !addTaskModalRef.current.contains(event.target)) {
        setShowAddTaskModal(false);
      }
      if (notificationModalRef.current && !notificationModalRef.current.contains(event.target)) {
        setShowNotificationModal(false);
      }
      if (alertModalRef.current && !alertModalRef.current.contains(event.target)) {
        setShowAlertModal(false); // Close alert if clicked outside
      }
    };

    document.addEventListener('mousedown', handleClickOutside);
    return () => {
      document.removeEventListener('mousedown', handleClickOutside);
    };
  }, []);

  // Function to show in-app notification
  const showAppNotification = (message) => {
    setNotificationMessage(message);
    setShowNotificationModal(true);
    setTimeout(() => {
      setShowNotificationModal(false);
      setNotificationMessage('');
    }, 5000); // Notification disappears after 5 seconds
  };

  // Custom Alert Modal function
  const showAlert = (message) => {
    setAlertMessage(message);
    setShowAlertModal(true);
  };

  // Check for task reminders and unfinished tasks
  useEffect(() => {
    if (!isAuthReady || !userId) return;

    const now = new Date();
    tasks.forEach(task => {
      // Task reminders (if reminder is set and within next 24 hours)
      if (task.reminder && task.reminder > now && task.reminder.getTime() - now.getTime() < 24 * 60 * 60 * 1000) {
        showAppNotification(`תזכורת: יש לך משימת ${task.type} - "${task.description}" היום!`);
      }
    });

    // Unfinished tasks for the current day (when viewing monthly calendar or todo list)
    // This notification will appear if there are unfinished tasks for today
    const today = new Date();
    const unfinishedTasksToday = tasks.filter(task =>
      !task.completed && task.startDate && task.startDate.toDateString() <= today.toDateString() && task.endDate.toDateString() >= today.toDateString()
    );
    if (unfinishedTasksToday.length > 0) {
      showAppNotification(`יש לך ${unfinishedTasksToday.length} משימות פעילות שלא סיימת לתאריך ${today.toLocaleDateString('he-IL')}.`);
    }

  }, [tasks, isAuthReady, userId]);

  // Event handlers for Firestore operations
  const handleAddTask = async (newTask) => {
    if (!db || !userId) {
      console.error("Firestore or User ID not available.");
      return;
    }
    try {
      await addDoc(collection(db, `artifacts/${appId}/users/${userId}/tasks`), {
        ...newTask,
        userId: userId, // Ensure userId is stored with the task
        startDate: Timestamp.fromDate(newTask.startDate), // Convert Date to Timestamp
        endDate: Timestamp.fromDate(newTask.endDate), // Convert Date to Timestamp
        reminder: newTask.reminder ? Timestamp.fromDate(newTask.reminder) : null // Convert Date to Timestamp for reminder
      });
      setShowAddTaskModal(false);
    } catch (e) {
      console.error("Error adding task: ", e);
    }
  };

  const handleToggleTaskComplete = async (id, completed) => {
    if (!db || !userId) return;
    try {
      await updateDoc(doc(db, `artifacts/${appId}/users/${userId}/tasks`, id), {
        completed: completed
      });
    } catch (e) {
      console.error("Error updating task: ", e);
    }
  };

  const handleDeleteTask = async (id) => {
    if (!db || !userId) return;
    try {
      await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/tasks`, id));
    } catch (e) {
      console.error("Error deleting task: ", e);
    }
  };

  // Helper to get color for task type
  const getTaskTypeColor = (type, mode = 'dot') => {
    if (mode === 'dot') {
      switch (type) {
        case 'לימודים': return 'bg-sky-400'; // תכלת
        case 'עבודה': return 'bg-purple-400'; // סגלגל
        case 'מילואים': return 'bg-emerald-400'; // ירוק מנטה
        case 'אחר': return 'bg-gray-400';
        default: return 'bg-gray-400';
      }
    } else if (mode === 'bg') { // Background color for list items
      switch (type) {
        case 'לימודים': return 'bg-sky-200'; // תכלת
        case 'עבודה': return 'bg-purple-200'; // סגלגל
        case 'מילואים': return 'bg-emerald-200'; // ירוק מנטה
        case 'אחר': return 'bg-gray-200';
        default: return 'bg-gray-200';
      }
    } else if (mode === 'border') { // Border color for list items
        switch (type) {
            case 'לימודים': return 'border-sky-400';
            case 'עבודה': return 'border-purple-400';
            case 'מילואים': return 'border-emerald-400';
            case 'אחר': return 'border-gray-400';
            default: return 'border-gray-400';
        }
    }
  };

  const getCourseColor = (category) => {
    return coursesData.find(c => c.category === category)?.color || 'bg-gray-400';
  };


  // Component for adding/editing tasks (now includes start/end dates and reminder)
  const AddTaskModal = ({ onClose, onSave, task = null }) => {
    const [description, setDescription] = useState(task ? task.description : '');
    const [type, setType] = useState(task ? task.type : 'לימודים');
    const [startDate, setStartDate] = useState(task ? task.startDate.toISOString().split('T')[0] : selectedDate.toISOString().split('T')[0]);
    const [endDate, setEndDate] = useState(task ? task.endDate.toISOString().split('T')[0] : selectedDate.toISOString().split('T')[0]);
    const [reminderMinutes, setReminderMinutes] = useState(task && task.reminder ? Math.round((task.startDate.getTime() - task.reminder.getTime()) / (1000 * 60)) : 0);

    const handleSubmit = (e) => {
      e.preventDefault();
      const newStartDate = new Date(startDate);
      const newEndDate = new Date(endDate);

      // Set times to start/end of day for consistency for multi-day tasks
      newStartDate.setHours(0, 0, 0, 0);
      newEndDate.setHours(23, 59, 59, 999);

      if (newStartDate > newEndDate) {
        showAlert('תאריך סיום חייב להיות אחרי תאריך התחלה.'); // Using custom alert
        return;
      }

      let reminderTime = null;
      if (reminderMinutes > 0) {
        // Reminder is relative to the start date (start of day)
        reminderTime = new Date(newStartDate.getTime() - reminderMinutes * 60 * 1000);
      }

      onSave({ description, type, completed: task ? task.completed : false, startDate: newStartDate, endDate: newEndDate, reminder: reminderTime });
    };

    return (
      <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
        <div ref={addTaskModalRef} className="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
          <h2 className="text-2xl font-bold mb-4 text-center text-pink-700">הוסף משימה חדשה</h2>
          <form onSubmit={handleSubmit} className="space-y-4">
            <div>
              <label htmlFor="taskDescription" className="block text-gray-700 text-sm font-bold mb-2">תיאור משימה:</label>
              <input
                type="text"
                id="taskDescription"
                className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-pink-300"
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                required
              />
            </div>
            <div>
              <label htmlFor="taskType" className="block text-gray-700 text-sm font-bold mb-2">סוג משימה:</label>
              <select
                id="taskType"
                className="shadow border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-pink-300"
                value={type}
                onChange={(e) => setType(e.target.value)}
              >
                <option value="לימודים">לימודים</option>
                <option value="עבודה">עבודה</option>
                <option value="מילואים">מילואים</option>
                <option value="אחר">אחר</option>
              </select>
            </div>
            <div>
              <label htmlFor="startDate" className="block text-gray-700 text-sm font-bold mb-2">תאריך התחלה:</label>
              <input
                type="date"
                id="startDate"
                className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-pink-300"
                value={startDate}
                onChange={(e) => setStartDate(e.target.value)}
                required
              />
            </div>
            <div>
              <label htmlFor="endDate" className="block text-gray-700 text-sm font-bold mb-2">תאריך סיום:</label>
              <input
                type="date"
                id="endDate"
                className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-pink-300"
                value={endDate}
                onChange={(e) => setEndDate(e.target.value)}
                required
              />
            </div>
            <div>
              <label htmlFor="reminder" className="block text-gray-700 text-sm font-bold mb-2">תזכורת מראש (דקות לפני תאריך ההתחלה):</label>
              <input
                type="number"
                id="reminder"
                className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-pink-300"
                value={reminderMinutes}
                onChange={(e) => setReminderMinutes(Number(e.target.value))}
                min="0"
              />
            </div>
            <div className="flex justify-end space-x-4">
              <button
                type="button"
                onClick={onClose}
                className="bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
              >
                ביטול
              </button>
              <button
                type="submit"
                className="bg-pink-500 hover:bg-pink-600 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
              >
                שמור משימה
              </button>
            </div>
          </form>
        </div>
      </div>
    );
  };

  // Monthly Calendar View
  const MonthlyCalendarView = () => {
    const [hoveredDayTasks, setHoveredDayTasks] = useState([]);
    const [hoveredDayPosition, setHoveredDayPosition] = useState({ x: 0, y: 0 });
    const [showTooltip, setShowTooltip] = useState(false);

    const getDaysInMonth = (year, month) => {
      return new Date(year, month + 1, 0).getDate();
    };

    const getFirstDayOfMonth = (year, month) => {
      return new Date(year, month, 1).getDay(); // 0 for Sunday, 1 for Monday
    };

    const year = selectedDate.getFullYear();
    const month = selectedDate.getMonth();
    const numDays = getDaysInMonth(year, month);
    const firstDay = getFirstDayOfMonth(year, month); // 0-6, where 0 is Sunday

    const days = [];
    for (let i = 0; i < firstDay; i++) {
      days.push(null); // Empty placeholders for days before the 1st
    }
    for (let i = 1; i <= numDays; i++) {
      days.push(i);
    }

    const handleDayHover = (day, event) => {
      if (day === null) return;
      const date = new Date(year, month, day);
      // Normalize date to start of day for comparison
      date.setHours(0, 0, 0, 0);

      const dayTasks = tasks.filter(task => {
        // Normalize task dates to start/end of day for comparison
        const taskStartDate = new Date(task.startDate);
        taskStartDate.setHours(0, 0, 0, 0);
        const taskEndDate = new Date(task.endDate);
        taskEndDate.setHours(23, 59, 59, 999);

        return task.startDate && task.endDate && date >= taskStartDate && date <= taskEndDate;
      });
      setHoveredDayTasks(dayTasks);
      // Position the tooltip relative to the cursor, adjusting for screen edges
      const xPos = event.clientX + 10;
      const yPos = event.clientY + 10;
      setHoveredDayPosition({ x: xPos, y: yPos });
      setShowTooltip(true);
    };

    const handleDayLeave = () => {
      setShowTooltip(false);
      setHoveredDayTasks([]);
    };

    const goToPreviousMonth = () => {
      setSelectedDate(prev => {
        const newDate = new Date(prev);
        newDate.setMonth(prev.getMonth() - 1);
        return newDate;
      });
    };

    const goToNextMonth = () => {
      setSelectedDate(prev => {
        const newDate = new Date(prev);
        newDate.setMonth(prev.getMonth() + 1);
        return newDate;
      });
    };

    return (
      <div className="p-4 flex flex-col items-center w-full">
        <h1 className="text-3xl font-bold mb-6 text-pink-700 text-center">לוח שנה חודשי</h1>
        <div className="flex items-center justify-center mb-6 space-x-4">
          <button
            onClick={goToPreviousMonth}
            className="bg-pink-400 hover:bg-pink-500 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
          >
            &lt; חודש קודם
          </button>
          <span className="text-xl font-semibold text-gray-800">
            {selectedDate.toLocaleDateString('he-IL', { year: 'numeric', month: 'long' })}
          </span>
          <button
            onClick={goToNextMonth}
            className="bg-pink-400 hover:bg-pink-500 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
          >
            חודש הבא &gt;
          </button>
        </div>

        <div className="w-full max-w-4xl bg-white rounded-lg shadow-lg p-6">
          <div className="grid grid-cols-7 gap-2 text-center font-bold text-pink-600 mb-4">
            <div>א'</div> {/* Sunday */}
            <div>ב'</div> {/* Monday */}
            <div>ג'</div>
            <div>ד'</div>
            <div>ה'</div>
            <div>ו'</div>
            <div>ש'</div> {/* Saturday */}
          </div>
          <div className="grid grid-cols-7 gap-2">
            {days.map((day, index) => {
              const currentDate = day ? new Date(year, month, day) : null;
              const dayTasks = currentDate ? tasks.filter(task => {
                const taskStartDate = new Date(task.startDate);
                taskStartDate.setHours(0, 0, 0, 0);
                const taskEndDate = new Date(task.endDate);
                taskEndDate.setHours(23, 59, 59, 999);
                return task.startDate && task.endDate && currentDate.setHours(0,0,0,0) >= taskStartDate && currentDate.setHours(0,0,0,0) <= taskEndDate;
              }) : [];
              const isToday = currentDate && currentDate.toDateString() === new Date().toDateString();

              return (
                <div
                  key={index}
                  className={`relative p-2 h-24 border rounded-lg flex flex-col items-center justify-between cursor-pointer transition duration-150 ease-in-out
                    ${day ? 'bg-pink-50 hover:bg-pink-100' : 'bg-gray-100'}
                    ${isToday ? 'border-pink-500 ring-2 ring-pink-300' : 'border-gray-200'}
                  `}
                  onMouseEnter={(e) => handleDayHover(day, e)}
                  onMouseLeave={handleDayLeave}
                >
                  <span className={`font-semibold ${isToday ? 'text-pink-700' : 'text-gray-800'}`}>
                    {day}
                  </span>
                  <div className="flex flex-wrap justify-center gap-1 mt-1">
                    {dayTasks.map((task, i) => (
                      <div
                        key={i}
                        className={`w-3 h-3 rounded-full ${getTaskTypeColor(task.type, 'dot')}`}
                        title={task.description} // Tooltip on dot hover
                      ></div>
                    ))}
                  </div>
                </div>
              );
            })}
          </div>
          <button
            onClick={() => setShowAddTaskModal(true)}
            className="mt-6 w-full bg-pink-500 hover:bg-pink-600 text-white font-bold py-3 px-6 rounded-full shadow-lg transition duration-300 ease-in-out transform hover:scale-105"
          >
            + הוסף משימה
          </button>
        </div>

        {showTooltip && hoveredDayTasks.length > 0 && (
          <div
            className="absolute bg-pink-700 text-white p-3 rounded-lg shadow-xl z-50 pointer-events-none"
            style={{ left: hoveredDayPosition.x, top: hoveredDayPosition.y }}
          >
            <h3 className="font-bold mb-2">משימות ליום זה:</h3>
            <ul className="list-disc list-inside space-y-1">
              {hoveredDayTasks.map(task => (
                <li key={task.id} className={`${task.completed ? 'line-through text-gray-300' : ''}`}>
                  {task.description} ({task.type}) - {task.startDate.toLocaleDateString('he-IL')} עד {task.endDate.toLocaleDateString('he-IL')}
                </li>
              ))}
            </ul>
          </div>
        )}
      </div>
    );
  };

  // To-Do List View
  const TodoListView = () => {
    // Sort tasks by date, then by completion status (incomplete first)
    const sortedTasks = [...tasks].sort((a, b) => {
      const dateComparison = a.startDate.getTime() - b.startDate.getTime();
      if (dateComparison !== 0) return dateComparison;
      return (a.completed === b.completed) ? 0 : a.completed ? 1 : -1;
    });

    return (
      <div className="p-4 flex flex-col items-center w-full">
        <h1 className="text-3xl font-bold mb-6 text-pink-700 text-center">רשימת משימות</h1>
        <div className="w-full max-w-2xl bg-white rounded-lg shadow-lg p-6">
          <h2 className="text-2xl font-bold mb-4 text-pink-600 text-center">המשימות שלך</h2>
          {sortedTasks.length === 0 ? (
            <p className="text-gray-600 text-center">אין משימות ברשימה.</p>
          ) : (
            <ul className="space-y-3">
              {sortedTasks.map(task => (
                <li
                  key={task.id}
                  className={`flex items-center justify-between p-3 rounded-lg shadow-sm transition duration-300 ease-in-out border-r-4
                    ${task.completed ? 'bg-gray-200 text-gray-500 line-through border-gray-400' : getTaskTypeColor(task.type, 'bg') + ' ' + getTaskTypeColor(task.type, 'border')}
                  `}
                >
                  <div className="flex items-center flex-1">
                    <button
                      onClick={() => handleToggleTaskComplete(task.id, !task.completed)}
                      className={`mr-3 p-1 rounded-full border-2 ${task.completed ? 'bg-green-500 border-green-500 text-white' : 'border-gray-400 text-transparent hover:bg-gray-300'}`}
                      title={task.completed ? "בטל סימון" : "סמן כהושלם"}
                    >
                      {task.completed && (
                        <svg xmlns="http://www.w3.org/2000/svg" className="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
                          <path fillRule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clipRule="evenodd" />
                        </svg>
                      )}
                    </button>
                    <div>
                      <p className="font-semibold">{task.description}</p>
                      <p className="text-sm">
                        {task.startDate.toLocaleDateString('he-IL')} {task.endDate.toLocaleDateString('he-IL') !== task.startDate.toLocaleDateString('he-IL') ? ` - ${task.endDate.toLocaleDateString('he-IL')}` : ''} ({task.type})
                      </p>
                    </div>
                  </div>
                  <button
                    onClick={() => handleDeleteTask(task.id)}
                    className="text-red-500 hover:text-red-700 transition duration-200"
                    title="מחק משימה"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                      <path fillRule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm6 0a1 1 0 11-2 0v6a1 1 0 112 0V8z" clipRule="evenodd" />
                    </svg>
                  </button>
                </li>
              ))}
            </ul>
          )}
          <button
            onClick={() => setShowAddTaskModal(true)}
            className="mt-6 w-full bg-pink-500 hover:bg-pink-600 text-white font-bold py-3 px-6 rounded-full shadow-lg transition duration-300 ease-in-out transform hover:scale-105"
          >
            + הוסף משימה
          </button>
        </div>
      </div>
    );
  };

  // Draggable Course Item (for initial drag from sidebar)
  const DraggableCourse = ({ category, color, defaultText, durationIntervals }) => {
    const [{ isDragging }, drag] = useDrag(() => ({
      type: ItemTypes.COURSE,
      item: { category, color, customText: defaultText, durationIntervals },
      collect: (monitor) => ({
        isDragging: !!monitor.isDragging(),
      }),
    }));

    return (
      <div
        ref={drag}
        className={`p-2 rounded-md shadow-sm text-white text-center cursor-grab ${color} ${isDragging ? 'opacity-50' : 'opacity-100'}`}
        style={{ marginBottom: '8px' }}
      >
        {defaultText}
      </div>
    );
  };

  // Droppable Time Slot & Draggable Scheduled Item
  const TimeSlot = ({ interval, currentSchedule, onDropItem, onUpdateItem, forceEdit = false }) => {
    const hour = Math.floor(interval / 4);
    const minute = (interval % 4) * 15;
    const timeLabel = `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`;

    const slotContent = currentSchedule[interval];
    
    // Determine the actual content and its starting interval if this is a continuation
    let mainBlockContent = null;
    let mainBlockStartInterval = interval;
    if (slotContent && slotContent.type === 'continuation') {
        mainBlockStartInterval = slotContent.originalInterval;
        mainBlockContent = currentSchedule[mainBlockStartInterval];
    } else {
        mainBlockContent = slotContent;
    }

    const bgColor = mainBlockContent ? getCourseColor(mainBlockContent.category) : 'bg-pink-50';

    const [isEditing, setIsEditing] = useState(false);
    const [editText, setEditText] = useState(mainBlockContent?.customText || '');
    const inputRef = useRef(null);

    // Drag for already scheduled items
    const [{ isDragging: isDraggingScheduled }, dragScheduled] = useDrag(() => ({
      type: ItemTypes.SCHEDULED_COURSE,
      item: { ...mainBlockContent, originalInterval: mainBlockStartInterval },
      canDrag: !!mainBlockContent && mainBlockStartInterval === interval, // Only drag the start of the block
      collect: (monitor) => ({
        isDraggingScheduled: !!monitor.isDragging(),
      }),
    }));

    // Drop for any course type (new or scheduled)
    const [{ isOver, canDrop }, drop] = useDrop(() => ({
      accept: [ItemTypes.COURSE, ItemTypes.SCHEDULED_COURSE],
      drop: (item, monitor) => {
          // If dropping a scheduled item, clear its original block first
          if (monitor.getItemType() === ItemTypes.SCHEDULED_COURSE) {
              onDropItem(item.originalInterval, null); // Clear old block
          }
          onDropItem(interval, { ...item, forceEdit: true }); // Drop new item, trigger edit
      },
      collect: (monitor) => ({
        isOver: !!monitor.isOver(),
        canDrop: !!monitor.canDrop(),
      }),
    }));

    // Effect to trigger edit mode immediately after a new drop
    useEffect(() => {
      if (mainBlockContent && mainBlockContent.forceEdit && inputRef.current) {
        setIsEditing(true);
        setEditText(''); // Clear text for new input
        inputRef.current.focus();

        // Immediately update Firestore to remove the forceEdit flag
        onUpdateItem(mainBlockStartInterval, {
          ...mainBlockContent,
          forceEdit: false // Remove the flag
        });
      }
    }, [mainBlockContent, mainBlockStartInterval, onUpdateItem]);

    // Update editText when slotContent changes (e.g., after drop or schedule load)
    useEffect(() => {
      setEditText(mainBlockContent?.customText || '');
    }, [mainBlockContent]);

    const handleTextChange = (e) => {
      setEditText(e.target.value);
    };

    const handleEditBlur = () => {
      setIsEditing(false);
      if (mainBlockContent && editText !== mainBlockContent.customText) {
          const newDuration = editText.startsWith('להשלים') ? 9 : (coursesData.find(c => c.category === mainBlockContent.category)?.durationIntervals || 4);
          onUpdateItem(mainBlockStartInterval, {
              ...mainBlockContent,
              customText: editText,
              durationIntervals: newDuration,
          });
      }
    };

    const handleKeyDown = (e) => {
      if (e.key === 'Enter') {
        e.target.blur();
      }
    };

    const handleDoubleClick = () => {
      setIsEditing(true);
      setEditText(''); // Clear text immediately on double click
      setTimeout(() => {
        if (inputRef.current) {
          inputRef.current.focus();
        }
      }, 0);
    };

    // Render only the starting cell of a block
    if (mainBlockContent && mainBlockStartInterval !== interval) {
        return null; // This is a continuation cell, don't render separately
    }

    // Calculate end time for display
    const blockDurationIntervals = mainBlockContent?.durationIntervals || 4;
    const endInterval = mainBlockStartInterval + blockDurationIntervals;
    const endHour = Math.floor(endInterval / 4);
    const endMinute = (endInterval % 4) * 15;
    const blockEndTime = `${String(endHour).padStart(2, '0')}:${String(endMinute).padStart(2, '0')}`;

    // Calculate height for the main block based on its duration
    const actualHeight = blockDurationIntervals * FIFTEEN_MIN_HEIGHT;

    return (
      <div
        ref={drop}
        className={`relative border-b border-r border-pink-200 p-2 flex flex-col justify-center text-center text-gray-700
          ${isOver && canDrop ? 'bg-green-100' : ''}
          ${bgColor}
        `}
        style={{ height: `${actualHeight}px`, gridRow: `span ${blockDurationIntervals}` }} // Span multiple rows
      >
        {mainBlockContent ? (
          <div
            ref={dragScheduled}
            className={`p-1 rounded-md text-white ${getCourseColor(mainBlockContent.category)} cursor-grab w-full h-full flex flex-col justify-center
              ${isDraggingScheduled ? 'opacity-50' : 'opacity-100'}
            `}
            onDoubleClick={handleDoubleClick} // Add double click here
          >
            {isEditing ? (
              <input
                type="text"
                ref={inputRef}
                value={editText}
                onChange={handleTextChange}
                onBlur={handleEditBlur}
                onKeyDown={handleKeyDown}
                className="w-full h-full bg-transparent text-white text-center focus:outline-none text-lg"
              />
            ) : (
              <>
                <span className="font-semibold text-lg">{mainBlockContent.customText}</span>
                <span className="text-sm opacity-90 block mt-1">{timeLabel}-{blockEndTime}</span>
              </>
            )}
          </div>
        ) : (
          <span className="text-sm text-gray-500">{timeLabel}</span> // Display hour if slot is empty
        )}
        {/* Clear button for a slot */}
        {mainBlockContent && !isEditing && (
          <button
            onClick={() => onDropItem(mainBlockStartInterval, null)} // Pass null to clear the slot
            className="absolute top-1 left-1 text-red-600 hover:text-red-800 text-xs opacity-75"
            title="נקה משבצת"
          >
            <svg xmlns="http://www.w3.org/2000/svg" className="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
              <path fillRule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm6 0a1 1 0 11-2 0v6a1 1 0 112 0V8z" clipRule="evenodd" />
            </svg>
          </button>
        )}
      </div>
    );
  };

  // Daily Study Planner View
  const DailyStudyPlanner = () => {
    const todayStr = selectedDate.toISOString().split('T')[0];
    const currentDaySchedule = dailyStudySchedule[todayStr] || {};

    // Constants for 15-minute intervals
    const START_HOUR = 9;
    const END_HOUR = 19; // 7 PM
    const START_INTERVAL = START_HOUR * 4; // 36
    const END_INTERVAL = END_HOUR * 4; // 76

    const handleDropItem = async (startInterval, item) => {
        if (!db || !userId) return;

        const newSchedule = { ...currentDaySchedule };
        const droppedDuration = item?.customText?.startsWith('להשלים') ? 9 : (item?.durationIntervals || 4);

        // Function to clear a block of slots
        const clearBlock = (blockStartInterval, duration) => {
            for (let i = 0; i < duration; i++) {
                delete newSchedule[blockStartInterval + i];
            }
        };

        if (item === null) { // Clear event
            const itemToClear = newSchedule[startInterval];
            if (itemToClear) {
                const clearedDuration = itemToClear.customText?.startsWith('להשלים') ? 9 : (itemToClear.durationIntervals || 4);
                clearBlock(startInterval, clearedDuration);
            }
        } else { // Regular or long task drop
            // Clear any overlapping existing items first
            const newEndInterval = startInterval + droppedDuration;

            for (let i = START_INTERVAL; i < END_INTERVAL; i++) {
                const existingItem = newSchedule[i];
                if (existingItem && existingItem.type !== 'continuation') { // Only consider start of blocks
                    const existingDuration = existingItem.customText?.startsWith('להשלים') ? 9 : (existingItem.durationIntervals || 4);
                    const existingEndInterval = i + existingDuration;

                    // Check for overlap
                    if (Math.max(startInterval, i) < Math.min(newEndInterval, existingEndInterval)) {
                        clearBlock(i, existingDuration);
                    }
                }
            }

            // Place the new item at its starting interval
            newSchedule[startInterval] = { ...item, durationIntervals: droppedDuration, forceEdit: true }; // Set forceEdit to true

            // Mark subsequent intervals as covered for long tasks
            for (let i = 1; i < droppedDuration; i++) {
                newSchedule[startInterval + i] = { type: 'continuation', originalInterval: startInterval };
            }
        }

        try {
            const docRef = doc(db, `artifacts/${appId}/users/${userId}/dailyStudySchedules`, todayStr);
            await setDoc(docRef, { schedule: newSchedule, userId: userId }, { merge: true });
        } catch (e) {
            console.error("Error updating daily study schedule: ", e);
        }
    };

    const handleUpdateItem = async (startInterval, updatedContent) => {
        if (!db || !userId) return;
        const newSchedule = { ...currentDaySchedule };

        // Determine new duration based on updated text
        const newDuration = updatedContent.customText.startsWith('להשלים') ? 9 : (coursesData.find(c => c.category === updatedContent.category)?.durationIntervals || 4);
        
        // If duration changes, clear old block and re-place
        if (newDuration !== newSchedule[startInterval]?.durationIntervals) {
            const oldDuration = newSchedule[startInterval]?.durationIntervals || 4;
            // Clear old block
            for (let i = 0; i < oldDuration; i++) {
                delete newSchedule[startInterval + i];
            }

            // Place new block
            newSchedule[startInterval] = { ...updatedContent, durationIntervals: newDuration };
            for (let i = 1; i < newDuration; i++) {
                newSchedule[startInterval + i] = { type: 'continuation', originalInterval: startInterval };
            }
        } else {
            newSchedule[startInterval] = updatedContent;
        }

        // Ensure forceEdit is removed after the update
        if (newSchedule[startInterval]) {
            delete newSchedule[startInterval].forceEdit;
        }

        try {
            const docRef = doc(db, `artifacts/${appId}/users/${userId}/dailyStudySchedules`, todayStr);
            await setDoc(docRef, { schedule: newSchedule, userId: userId }, { merge: true });
        } catch (e) {
            console.error("Error updating item content: ", e);
        }
    };

    const scheduleCells = [];
    let currentInterval = START_INTERVAL;

    while (currentInterval < END_INTERVAL) {
      const hour = Math.floor(currentInterval / 4);
      const minute = (currentInterval % 4) * 15;
      const timeLabel = `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`;

      const slotContent = currentDaySchedule[currentInterval];
      let mainBlockContent = null;
      let blockDurationIntervals = 4; // Default to 1 hour

      if (slotContent && slotContent.type !== 'continuation') {
        mainBlockContent = slotContent;
        blockDurationIntervals = mainBlockContent.customText?.startsWith('להשלים') ? 9 : (mainBlockContent.durationIntervals || 4);
      } else if (slotContent && slotContent.type === 'continuation') {
        // This interval is part of a larger block, skip rendering a new TimeSlot
        currentInterval++;
        continue;
      }

      // Time Label (always render for each hour, but make it span)
      scheduleCells.push(
        <div
          key={`time-label-${currentInterval}`}
          className="border-b border-r border-pink-200 bg-pink-100 text-pink-700 font-semibold flex items-center justify-center"
          style={{
            gridRow: `span ${blockDurationIntervals}`, // Time label spans the block's duration
            height: `${blockDurationIntervals * FIFTEEN_MIN_HEIGHT}px`, // Explicit height
          }}
        >
          {timeLabel}
        </div>
      );

      // Content Slot (TimeSlot component)
      scheduleCells.push(
        <TimeSlot
          key={`content-slot-${currentInterval}`}
          interval={currentInterval}
          currentSchedule={currentDaySchedule}
          onDropItem={handleDropItem}
          onUpdateItem={handleUpdateItem}
          forceEdit={mainBlockContent?.forceEdit || false}
        />
      );

      currentInterval += blockDurationIntervals; // Move to the next available interval
    }

    return (
      <DndProvider backend={HTML5Backend}>
        <div className="p-4 flex flex-col md:flex-row items-start justify-center w-full">
          {/* Daily Schedule */}
          <div className="w-full md:w-3/5 lg:w-2/3 bg-white rounded-lg shadow-lg p-6 mb-6 md:mb-0 md:ml-6">
            <div className="flex items-center justify-between mb-6">
                <button
                    onClick={() => setSelectedDate(prev => {
                        const newDate = new Date(prev);
                        newDate.setDate(prev.getDate() - 1);
                        return newDate;
                    })}
                    className="bg-pink-400 hover:bg-pink-500 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out"
                >
                    &lt; יום קודם
                </button>
                <h1 className="text-3xl font-bold text-pink-700 text-center">
                    ניהול יום לימודים - {selectedDate.toLocaleDateString('he-IL', { weekday: 'long', day: 'numeric', month: 'numeric' })}
                </h1>
                <button
                    onClick={() => setSelectedDate(prev => {
                        const newDate = new Date(prev);
                        newDate.setDate(prev.getDate() + 1);
                        return newDate;
                    })}
                    className="bg-pink-400 hover:bg-pink-500 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out"
                >
                    יום הבא &gt;
                </button>
            </div>
            
            {/* The grid now represents 15-minute intervals */}
            <div className="grid grid-cols-[80px_1fr] auto-rows-[15px] border-l border-t border-pink-200">
              {scheduleCells}
            </div>
          </div>

          {/* Draggable Courses */}
          <div className="w-full md:w-2/5 lg:w-1/3 bg-white rounded-lg shadow-lg p-6">
            {coursesData.map(category => (
              <div key={category.category} className="mb-6">
                <div className="space-y-2">
                  <DraggableCourse
                    key={category.category}
                    category={category.category}
                    color={category.color}
                    defaultText={category.defaultText}
                    durationIntervals={category.durationIntervals}
                  />
                </div>
              </div>
            ))}
          </div>
        </div>
      </DndProvider>
    );
  };


  return (
    <div className="min-h-screen bg-pink-50 font-sans text-right" dir="rtl">
      {/* Tailwind CSS CDN */}
      <script src="https://cdn.tailwindcss.com"></script>
      {/* Font Inter */}
      <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet" />
      <style>
        {`
          body {
            font-family: 'Inter', sans-serif;
          }
          /* Custom scrollbar for better aesthetics */
          ::-webkit-scrollbar {
            width: 8px;
          }
          ::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 10px;
          }
          ::-webkit-scrollbar-thumb {
            background: #fbcfe8; /* pink-200 */
            border-radius: 10px;
          }
          ::-webkit-scrollbar-thumb:hover {
            background: #f9a8d4; /* pink-300 */
          }
          /* Keyframe for fade-in-up animation */
          @keyframes fade-in-up {
            from {
              opacity: 0;
              transform: translateY(20px);
            }
            to {
              opacity: 1;
              transform: translateY(0);
            }
          }
          .animate-fade-in-up {
            animation: fade-in-up 0.5s ease-out forwards;
          }
          /* Keyframe for alert fade-in */
          @keyframes alert-fade-in {
            from { opacity: 0; transform: scale(0.9); }
            to { opacity: 1; transform: scale(1); }
          }
          .alert-modal-animation {
            animation: alert-fade-in 0.3s ease-out forwards;
          }
        `}
      </style>

      {/* Navigation Bar */}
      <nav className="bg-pink-200 p-4 shadow-md sticky top-0 z-40">
        <div className="container mx-auto flex justify-between items-center flex-wrap">
          <div className="text-pink-800 text-2xl font-bold mb-2 md:mb-0">
            ניהול זמן
          </div>
          <div className="flex flex-wrap justify-center md:justify-end space-x-2 space-x-reverse">
            <button
              onClick={() => setCurrentPage('monthly')}
              className={`py-2 px-4 rounded-full font-semibold transition duration-300 ${currentPage === 'monthly' ? 'bg-pink-600 text-white shadow-lg' : 'bg-pink-300 text-pink-800 hover:bg-pink-400'}`}
            >
              לוח שנה חודשי
            </button>
            <button
              onClick={() => setCurrentPage('todo')}
              className={`py-2 px-4 rounded-full font-semibold transition duration-300 ${currentPage === 'todo' ? 'bg-pink-600 text-white shadow-lg' : 'bg-pink-300 text-pink-800 hover:bg-pink-400'}`}
            >
              רשימת משימות
            </button>
            <button
              onClick={() => setCurrentPage('study-planner')}
              className={`py-2 px-4 rounded-full font-semibold transition duration-300 ${currentPage === 'study-planner' ? 'bg-pink-600 text-white shadow-lg' : 'bg-pink-300 text-pink-800 hover:bg-pink-400'}`}
            >
              ניהול יום לימודים
            </button>
          </div>
        </div>
      </nav>

      {/* User ID Display */}
      {userId && (
        <div className="bg-pink-100 text-pink-700 text-sm p-2 text-center shadow-inner">
          מזהה משתמש: <span className="font-mono text-xs break-all">{userId}</span>
        </div>
      )}

      <main className="container mx-auto py-8">
        {currentPage === 'monthly' && <MonthlyCalendarView />}
        {currentPage === 'todo' && <TodoListView />}
        {currentPage === 'study-planner' && <DailyStudyPlanner />}
      </main>

      {/* Modals */}
      {showAddTaskModal && <AddTaskModal onClose={() => setShowAddTaskModal(false)} onSave={handleAddTask} />}

      {/* Notification Modal */}
      {showNotificationModal && (
        <div className="fixed bottom-4 right-4 bg-pink-700 text-white p-4 rounded-lg shadow-xl z-50 animate-fade-in-up" ref={notificationModalRef}>
          <p className="font-semibold">{notificationMessage}</p>
        </div>
      )}

      {/* Custom Alert Modal */}
      {showAlertModal && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
          <div ref={alertModalRef} className="bg-white p-6 rounded-lg shadow-xl w-full max-w-sm text-center alert-modal-animation">
            <h3 className="text-xl font-bold mb-4 text-pink-700">הודעה</h3>
            <p className="text-gray-700 mb-6">{alertMessage}</p>
            <button
              onClick={() => setShowAlertModal(false)}
              className="bg-pink-500 hover:bg-pink-600 text-white font-bold py-2 px-4 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
            >
              אישור
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

export default App;
