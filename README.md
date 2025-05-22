import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut, onAuthStateChanged, signInWithCustomToken, signInAnonymously } from 'firebase/auth';
import { getFirestore, doc, getDoc, setDoc, collection, query, where, onSnapshot, updateDoc, addDoc, deleteDoc } from 'firebase/firestore';

// Global variables provided by the Canvas environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Context for Auth and Firestore
const AppContext = createContext(null);

// Utility function to format date
const formatDate = (timestamp) => {
  if (!timestamp) return 'N/A';
  const date = timestamp.toDate ? timestamp.toDate() : new Date(timestamp);
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  });
};

// Custom Modal Component
const Modal = ({ isOpen, onClose, title, children }) => {
  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-black bg-opacity-85 flex items-center justify-center z-50 p-4">
      <div className="bg-gray-950 p-6 rounded-xl shadow-2xl w-full max-w-lg border-2 border-pink-700 animate-scale-in">
        <div className="flex justify-between items-center mb-4">
          <h2 className="text-2xl font-bold text-pink-400 drop-shadow-pink">{title}</h2>
          <button onClick={onClose} className="text-pink-500 hover:text-white text-3xl leading-none transition-colors duration-200">&times;</button>
        </div>
        <div className="max-h-[80vh] overflow-y-auto custom-scrollbar pr-2">
          {children}
        </div>
      </div>
    </div>
  );
};

// Loading Spinner Component
const LoadingSpinner = () => (
  <div className="flex justify-center items-center h-full">
    <div className="animate-spin rounded-full h-16 w-16 border-t-4 border-b-4 border-pink-400 border-opacity-75"></div>
  </div>
);

// Helper to format Firebase errors for user display
const formatFirebaseError = (error) => {
  if (error.code === 'permission-denied') {
    return 'Permission Denied: Please ensure your Firebase Firestore Security Rules are correctly configured to allow this operation. Check the "Rules" tab in your Firebase Console.';
  }
  return error.message || 'An unknown error occurred.';
};

// App Component
const App = () => {
  const [app, setApp] = useState(null);
  const [auth, setAuth] = useState(null);
  const [db, setDb] = useState(null);
  const [currentUser, setCurrentUser] = useState(null);
  const [userId, setUserId] = useState(null);
  const [loading, setLoading] = useState(true);
  const [currentPage, setCurrentPage] = useState('auth'); // 'auth', 'landing', 'project', 'admin'
  const [selectedProject, setSelectedProject] = useState(null);
  const [showNotification, setShowNotification] = useState(false);
  const [notificationMessage, setNotificationMessage] = useState('');

  // Initialize Firebase and handle authentication
  useEffect(() => {
    const initializeFirebase = async () => {
      try {
        const firebaseApp = initializeApp(firebaseConfig);
        const firebaseAuth = getAuth(firebaseApp);
        const firestoreDb = getFirestore(firebaseApp);

        setApp(firebaseApp);
        setAuth(firebaseAuth);
        setDb(firestoreDb);

        // Sign in with custom token if available, otherwise anonymously
        if (initialAuthToken) {
          await signInWithCustomToken(firebaseAuth, initialAuthToken);
        } else {
          await signInAnonymously(firebaseAuth);
        }

        // Listen for auth state changes
        const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
          if (user) {
            setCurrentUser(user);
            setUserId(user.uid);
            // Check user profile in Firestore
            const userDocRef = doc(firestoreDb, `artifacts/${appId}/users/${user.uid}/profile/data`);
            const userDocSnap = await getDoc(userDocRef);

            if (userDocSnap.exists()) {
              const userData = userDocSnap.data();
              if (userData.approved) {
                setCurrentPage('landing');
              } else {
                setCurrentPage('auth'); // Stay on auth page, show pending message
                showAppNotification('Your account is pending approval from the master account.');
              }
            } else {
              // New user, mark as pending approval
              await setDoc(userDocRef, {
                uid: user.uid,
                email: user.email,
                displayName: user.displayName || user.email,
                approved: false,
                role: 'user',
                createdAt: new Date(),
              });
              setCurrentPage('auth'); // Stay on auth page, show pending message
              showAppNotification('Your account has been created and is pending approval from the master account.');
            }
          } else {
            setCurrentUser(null);
            setUserId(null);
            setCurrentPage('auth');
          }
          setLoading(false);
        });

        return () => unsubscribe();
      } catch (error) {
        console.error("Error initializing Firebase or signing in:", error);
        showAppNotification(`Error initializing Firebase or signing in: ${formatFirebaseError(error)}`);
        setLoading(false);
      }
    };

    initializeFirebase();
  }, [appId, firebaseConfig, initialAuthToken]);

  const showAppNotification = (message) => {
    setNotificationMessage(message);
    setShowNotification(true);
    setTimeout(() => setShowNotification(false), 5000); // Hide after 5 seconds
  };

  const handleGoogleSignIn = async () => {
    if (!auth || !db) {
      showAppNotification("Firebase not initialized.");
      return;
    }
    const provider = new GoogleAuthProvider();
    try {
      setLoading(true);
      const result = await signInWithPopup(auth, provider);
      const user = result.user;

      const userDocRef = doc(db, `artifacts/${appId}/users/${user.uid}/profile/data`);
      const userDocSnap = await getDoc(userDocRef);

      if (userDocSnap.exists()) {
        const userData = userDocSnap.data();
        if (userData.approved) {
          setCurrentPage('landing');
        } else {
          showAppNotification('Your account is pending approval from the master account.');
        }
      } else {
        // New user, mark as pending approval
        await setDoc(userDocRef, {
          uid: user.uid,
          email: user.email,
          displayName: user.displayName || user.email,
          approved: false,
          role: 'user',
          createdAt: new Date(),
        });
        showAppNotification('Your account has been created and is pending approval from the master account.');
      }
    } catch (error) {
      console.error("Error during Google Sign-In:", error);
      showAppNotification(`Google Sign-In Error: ${formatFirebaseError(error)}`);
    } finally {
      setLoading(false);
    }
  };

  const handleSignOut = async () => {
    if (auth) {
      try {
        await signOut(auth);
        setCurrentPage('auth');
        setCurrentUser(null);
        setUserId(null);
        setSelectedProject(null);
        showAppNotification('You have been signed out.');
      } catch (error) {
              console.error("Error signing out:", error);
              showAppNotification(`Sign Out Error: ${formatFirebaseError(error)}`);
            }
          }
        };

  const handleMasterLogin = async (username, password) => {
    if (!auth || !db) {
      showAppNotification("Firebase not initialized.");
      return;
    }
    if (username === 'Mayau' && password === 'MayauOffice!') {
      try {
        // Simulate master login by creating a custom user or using a predefined master UID
        // For this demo, we'll just set a flag and redirect to admin page
        // In a real app, this would involve a secure backend authentication.
        // For now, we'll just set the current user as master for the session.
        // This is a client-side simulation and not secure for production.
        const masterUserDocRef = doc(db, `artifacts/${appId}/users/master_mayau_account/profile/data`);
        await setDoc(masterUserDocRef, {
          uid: 'master_mayau_account',
          email: 'master@mayau.com',
          displayName: 'Mayau Master',
          approved: true,
          role: 'master',
          createdAt: new Date(),
        }, { merge: true });

        // Temporarily set current user to simulate master access
        setCurrentUser({
          uid: 'master_mayau_account',
          email: 'master@mayau.com',
          displayName: 'Mayau Master',
        });
        setUserId('master_mayau_account');
        setCurrentPage('admin');
        showAppNotification('Master account logged in successfully.');
      } catch (error) {
        console.error("Error simulating master login:", error);
        showAppNotification(`Master Login Error: ${formatFirebaseError(error)}`);
      }
    } else {
      showAppNotification('Invalid master username or password.');
    }
  };

  const currentYear = new Date().getFullYear();
  const appVersion = '1.0.0'; // Example version number

  if (loading) {
    return (
      <div className="min-h-screen bg-gray-950 text-gray-200 flex items-center justify-center">
        <LoadingSpinner />
      </div>
    );
  }

  return (
    <AppContext.Provider value={{ auth, db, currentUser, userId, appId, showAppNotification }}>
      <div className="min-h-screen bg-gray-950 text-gray-200 flex flex-col font-inter">
        {/* Notification Banner */}
        {showNotification && (
          <div className="fixed top-0 left-0 right-0 bg-pink-600 text-white text-center p-3 z-50 shadow-lg animate-fade-in-down">
            {notificationMessage}
          </div>
        )}

        {/* Header */}
        <header className="bg-black p-4 shadow-lg flex items-center justify-between border-b border-pink-700">
          <div className="flex items-center">
            {/* Logo Placeholder */}
            <div className="text-pink-400 text-3xl font-bold mr-4 drop-shadow-pink">
              <span role="img" aria-label="Mayau Logo">🐱</span>
            </div>
            <h1 className="text-2xl font-bold text-white hidden sm:block">Mayau - Designed by Dr. Vin</h1>
          </div>
          {/* Removed Mayau SDN BHD from the center of the header */}
          <div className="flex items-center space-x-4">
            {currentUser && currentUser.uid && currentUser.uid !== 'master_mayau_account' && (
              <span className="text-sm text-gray-400 hidden md:block">
                Logged in as: <span className="text-pink-400 drop-shadow-pink">{currentUser.displayName || currentUser.email}</span>
              </span>
            )}
            {currentUser && (
              <button
                onClick={handleSignOut}
                className="bg-red-700 hover:bg-red-800 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-red-900"
              >
                Sign Out
              </button>
            )}
          </div>
        </header>

        <main className="flex-grow p-4 md:p-8 flex items-center justify-center">
          {currentPage === 'auth' && (
            <AuthPage
              onGoogleSignIn={handleGoogleSignIn}
              onMasterLogin={handleMasterLogin}
              currentUser={currentUser}
              showAppNotification={showAppNotification}
            />
          )}
          {currentPage === 'landing' && currentUser && (
            <LandingPage onSelectProject={setSelectedProject} setCurrentPage={setCurrentPage} />
          )}
          {currentPage === 'project' && selectedProject && currentUser && (
            <ProjectDashboard projectType={selectedProject} setCurrentPage={setCurrentPage} />
          )}
          {currentPage === 'admin' && currentUser && userId === 'master_mayau_account' && (
            <AdminApprovalPage setCurrentPage={setCurrentPage} />
          )}
        </main>

        {/* Footer */}
        <footer className="bg-black p-4 text-center text-gray-400 text-sm border-t border-pink-700">
          Coded by Dr. Vin ({currentYear}) (Version {appVersion}) for Mayau SDN BHD
        </footer>
      </div>
    </AppContext.Provider>
  );
};

// Auth Page Component
const AuthPage = ({ onGoogleSignIn, onMasterLogin, currentUser, showAppNotification }) => {
  const [masterUsername, setMasterUsername] = useState('');
  const [masterPassword, setMasterPassword] = useState('');

  const handleMasterSubmit = (e) => {
    e.preventDefault();
    onMasterLogin(masterUsername, masterPassword);
  };

  return (
    <div className="flex flex-col items-center justify-center p-6 bg-gray-950 rounded-xl shadow-2xl border-2 border-pink-700 w-full max-w-md animate-fade-in">
      <h2 className="text-3xl font-bold text-pink-400 mb-6 drop-shadow-pink">Welcome to Mayau</h2>
      {currentUser && !currentUser.emailVerified && currentUser.uid !== 'master_mayau_account' && (
        <p className="text-yellow-400 mb-4 text-center">
          Your account is pending approval from the master account. Please wait for an administrator to approve your access.
        </p>
      )}
      <button
        onClick={onGoogleSignIn}
        className="flex items-center justify-center bg-blue-700 hover:bg-blue-800 text-white font-semibold py-3 px-6 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 w-full mb-6 border border-blue-900"
      >
        <svg className="w-6 h-6 mr-3" viewBox="0 0 24 24" fill="currentColor">
          <path d="M12.24 10.27c-.36-.45-.6-.94-.6-1.5 0-1.44 1.2-2.6 2.6-2.6 1.44 0 2.6 1.2 2.6 2.6 0 .56-.24 1.05-.6 1.5l.6 1.2c.72-.96 1.2-2.16 1.2-3.48C18 6.12 15.88 4 13.2 4 10.52 4 8.4 6.12 8.4 8.8c0 1.32.48 2.52 1.2 3.48l.6 1.2c-.36.45-.6.94-.6 1.5 0 1.44 1.2 2.6 2.6 2.6 1.44 0 2.6-1.2 2.6-2.6 0-.56-.24-1.05-.6-1.5l.6-1.2c.72.96 1.2 2.16 1.2 3.48C18 17.88 15.88 20 13.2 20 10.52 20 8.4 17.88 8.4 15.2c0-1.32.48-2.52 1.2-3.48L12.24 10.27zM22 12c0-5.52-4.48-10-10-10S2 6.48 2 12s4.48 10 10 10 10-4.48 10-10zM12 18c-3.32 0-6-2.68-6-6s2.68-6 6-6 6 2.68 6 6-2.68 6-6 6z" />
        </svg>
        Sign in with Google
      </button>

      <div className="w-full text-center text-gray-600 mb-6 font-mono text-lg">--- ACCESS PORT ---</div>
      <form onSubmit={handleMasterSubmit} className="w-full">
        <h3 className="text-xl font-semibold text-pink-400 mb-4 drop-shadow-pink">Master Account Login</h3>
        <div className="mb-4">
          <label className="block text-gray-300 text-sm font-bold mb-2" htmlFor="master-username">
            Username
          </label>
          <input
            type="text"
            id="master-username"
            className="shadow-inner appearance-none border border-pink-700 rounded-lg w-full py-2 px-3 text-pink-200 leading-tight focus:outline-none focus:ring-2 focus:ring-pink-500 bg-gray-900 placeholder-gray-500"
            value={masterUsername}
            onChange={(e) => setMasterUsername(e.target.value)}
            required
          />
        </div>
        <div className="mb-6">
          <label className="block text-gray-300 text-sm font-bold mb-2" htmlFor="master-password">
            Password
          </label>
          <input
            type="password"
            id="master-password"
            className="shadow-inner appearance-none border border-pink-700 rounded-lg w-full py-2 px-3 text-pink-200 mb-3 leading-tight focus:outline-none focus:ring-2 focus:ring-pink-500 bg-gray-900 placeholder-gray-500"
            value={masterPassword}
            onChange={(e) => setMasterPassword(e.target.value)}
            required
          />
        </div>
        <button
          type="submit"
          className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-3 px-6 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 w-full border border-pink-900"
        >
          Login as Master
        </button>
      </form>
    </div>
  );
};

// Admin Approval Page Component
const AdminApprovalPage = ({ setCurrentPage }) => {
  const { db, appId, showAppNotification } = useContext(AppContext);
  const [pendingUsers, setPendingUsers] = useState([]);
  const [loadingUsers, setLoadingUsers] = useState(true);

  useEffect(() => {
    if (!db) return;

    const usersCollectionRef = collection(db, `artifacts/${appId}/users`);
    const q = query(usersCollectionRef, where('profile.data.approved', '==', false));

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const users = [];
      snapshot.forEach(doc => {
        const userData = doc.data().profile?.data; // Access nested profile.data
        if (userData) {
          users.push({ id: doc.id, ...userData });
        }
      });
      setPendingUsers(users);
      setLoadingUsers(false);
    }, (error) => {
      console.error("Error fetching pending users:", error);
      showAppNotification(`Error fetching pending users: ${formatFirebaseError(error)}`);
      setLoadingUsers(false);
    });

    return () => unsubscribe();
  }, [db, appId, showAppNotification]);

  const handleApproveUser = async (userUid) => {
    if (!db) return;
    try {
      const userDocRef = doc(db, `artifacts/${appId}/users/${userUid}/profile/data`);
      await updateDoc(userDocRef, { approved: true });
      showAppNotification(`User ${userUid} approved successfully.`);
    } catch (error) {
      console.error("Error approving user:", error);
      showAppNotification(`Error approving user: ${formatFirebaseError(error)}`);
    }
  };

  if (loadingUsers) return <LoadingSpinner />;

  return (
    <div className="flex flex-col items-center p-6 bg-gray-950 rounded-xl shadow-2xl border-2 border-pink-700 w-full max-w-4xl animate-fade-in">
      <h2 className="text-3xl font-bold text-pink-400 mb-6 drop-shadow-pink">Master Account - User Approval</h2>
      <button
        onClick={() => setCurrentPage('landing')}
        className="mb-4 bg-gray-800 hover:bg-gray-900 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
      >
        &larr; Go to Landing Page
      </button>

      {pendingUsers.length === 0 ? (
        <p className="text-gray-400">No pending users for approval.</p>
      ) : (
        <div className="w-full">
          {pendingUsers.map((user) => (
            <div key={user.id} className="flex items-center justify-between bg-gray-900 p-4 rounded-lg mb-3 border border-gray-800 shadow-md">
              <div>
                <p className="text-lg text-white">{user.displayName || user.email}</p>
                <p className="text-sm text-gray-400">UID: <span className="font-mono text-pink-300">{user.uid}</span></p>
                <p className="text-sm text-gray-400">Email: {user.email}</p>
              </div>
              <button
                onClick={() => handleApproveUser(user.id)}
                className="bg-green-700 hover:bg-green-800 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-green-900"
              >
                Approve
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

// Landing Page Component
const LandingPage = ({ onSelectProject, setCurrentPage }) => {
  const { currentUser, userId } = useContext(AppContext);

  // Check if the current user is the master account
  const isMasterAccount = userId === 'master_mayau_account';

  return (
    <div className="flex flex-col items-center justify-center p-6 bg-gray-950 rounded-xl shadow-2xl border-2 border-pink-700 w-full max-w-4xl animate-fade-in">
      <h2 className="text-3xl font-bold text-pink-400 mb-8 drop-shadow-pink">Select Your Workspace</h2>

      {isMasterAccount && (
        <button
          onClick={() => setCurrentPage('admin')}
          className="mb-8 bg-pink-700 hover:bg-pink-800 text-white font-semibold py-3 px-6 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
        >
          Go to Master Approval Page
        </button>
      )}

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 w-full">
        <ProjectCard
          title="Mayau Office"
          description="Manage administrative tasks and general office operations."
          icon="🏢"
          onClick={() => { onSelectProject('Mayau Office'); setCurrentPage('project'); }}
        />
        <ProjectCard
          title="Mayau Studios"
          description="Oversee creative projects, media production, and artistic endeavors."
          icon="🎬"
          onClick={() => { onSelectProject('Mayau Studios'); setCurrentPage('project'); }}
        />
        <ProjectCard
          title="General Combined Tasks"
          description="Handle miscellaneous tasks and cross-departmental projects."
          icon="🔗"
          onClick={() => { onSelectProject('General Combined Tasks'); setCurrentPage('project'); }}
        />
      </div>
    </div>
  );
};

// Project Card Component
const ProjectCard = ({ title, description, icon, onClick }) => (
  <div
    onClick={onClick}
    className="bg-gray-900 p-6 rounded-xl shadow-lg cursor-pointer hover:bg-gray-800 transition duration-300 ease-in-out transform hover:scale-105 border border-pink-600 flex flex-col items-center text-center"
  >
    <div className="text-5xl mb-4 text-pink-400 drop-shadow-pink">{icon}</div>
    <h3 className="text-2xl font-semibold text-pink-400 mb-2 drop-shadow-pink">{title}</h3>
    <p className="text-gray-300 text-sm">{description}</p>
  </div>
);

// Project Dashboard Component
const ProjectDashboard = ({ projectType, setCurrentPage }) => {
  const { db, userId, appId, showAppNotification } = useContext(AppContext);
  const [project, setProject] = useState(null);
  const [tasks, setTasks] = useState([]);
  const [allUsers, setAllUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [isTaskModalOpen, setIsTaskModalOpen] = useState(false);
  const [editingTask, setEditingTask] = useState(null);
  const [isTeamModalOpen, setIsTeamModalOpen] = useState(false);
  const [selectedTaskForDetail, setSelectedTaskForDetail] = useState(null);
  const [isTaskDetailModalOpen, setIsTaskDetailModalOpen] = useState(false);
  const [filterStatus, setFilterStatus] = useState('all'); // 'all', 'pending', 'in-progress', 'completed'

  // Fetch project details and tasks
  useEffect(() => {
    if (!db || !projectType) return;

    const projectCollectionRef = collection(db, `artifacts/${appId}/public/data/projects`);
    const q = query(projectCollectionRef, where('name', '==', projectType));

    const unsubscribeProject = onSnapshot(q, async (snapshot) => {
      if (!snapshot.empty) {
        const projectDoc = snapshot.docs[0];
        setProject({ id: projectDoc.id, ...projectDoc.data() });
      } else {
        // If project doesn't exist, create it
        const newProjectRef = await addDoc(projectCollectionRef, {
          name: projectType,
          type: projectType.toLowerCase().replace(' ', '-'),
          members: [], // Initialize with no members
        });
        setProject({ id: newProjectRef.id, name: projectType, type: projectType.toLowerCase().replace(' ', '-'), members: [] });
      }
      setLoading(false);
    }, (error) => {
      console.error("Error fetching project:", error);
      showAppNotification(`Error fetching project data: ${formatFirebaseError(error)}`);
      setLoading(false);
    });

    return () => unsubscribeProject();
  }, [db, projectType, appId, showAppNotification]);

  // Fetch tasks for the selected project
  useEffect(() => {
    if (!db || !project) return;

    const tasksCollectionRef = collection(db, `artifacts/${appId}/public/data/tasks`);
    const q = query(tasksCollectionRef, where('projectId', '==', project.id));

    const unsubscribeTasks = onSnapshot(q, (snapshot) => {
      const fetchedTasks = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setTasks(fetchedTasks);
    }, (error) => {
      console.error("Error fetching tasks:", error);
      showAppNotification(`Error fetching tasks: ${formatFirebaseError(error)}`);
    });

    return () => unsubscribeTasks();
  }, [db, project, appId, showAppNotification]);

  // Fetch all users for team management and task assignment
  useEffect(() => {
    if (!db) return;

    const usersCollectionRef = collection(db, `artifacts/${appId}/users`);
    const q = query(usersCollectionRef, where('profile.data.approved', '==', true));

    const unsubscribeUsers = onSnapshot(q, (snapshot) => {
      const users = [];
      snapshot.forEach(doc => {
        const userData = doc.data().profile?.data;
        if (userData) {
          users.push({ uid: doc.id, ...userData });
        }
      });
      setAllUsers(users);
    }, (error) => {
      console.error("Error fetching all users:", error);
      showAppNotification(`Error fetching users for team management: ${formatFirebaseError(error)}`);
    });

    return () => unsubscribeUsers();
  }, [db, appId, showAppNotification]);

  const handleCreateTask = () => {
    setEditingTask(null);
    setIsTaskModalOpen(true);
  };

  const handleEditTask = (task) => {
    setEditingTask(task);
    setIsTaskModalOpen(true);
  };

  const handleDeleteTask = async (taskId) => {
    if (!db) return;
    try {
      // Show confirmation modal instead of alert
      const confirmed = window.confirm("Are you sure you want to delete this task?");
      if (confirmed) {
        await deleteDoc(doc(db, `artifacts/${appId}/public/data/tasks`, taskId));
        showAppNotification('Task deleted successfully!');
      }
    } catch (error) {
      console.error("Error deleting task:", error);
      showAppNotification(`Error deleting task: ${formatFirebaseError(error)}`);
    }
  };

  const handleSaveTask = async (taskData) => {
    if (!db || !project) return;
    try {
      if (editingTask) {
        // Update existing task
        await updateDoc(doc(db, `artifacts/${appId}/public/data/tasks`, editingTask.id), taskData);
        showAppNotification('Task updated successfully!');
      } else {
        // Add new task
        await addDoc(collection(db, `artifacts/${appId}/public/data/tasks`), {
          ...taskData,
          projectId: project.id,
          createdAt: new Date(),
        });
        showAppNotification('Task created successfully!');
      }
      setIsTaskModalOpen(false);
      setEditingTask(null);
    } catch (error) {
      console.error("Error saving task:", error);
      showAppNotification(`Error saving task: ${formatFirebaseError(error)}`);
    }
  };

  const handleViewTaskDetail = (task) => {
    setSelectedTaskForDetail(task);
    setIsTaskDetailModalOpen(true);
  };

  const filteredTasks = tasks.filter(task => {
    if (filterStatus === 'all') return true;
    return task.status === filterStatus;
  });

  if (loading || !project) {
    return <LoadingSpinner />;
  }

  return (
    <div className="flex flex-col p-6 bg-gray-950 rounded-xl shadow-2xl border-2 border-pink-700 w-full max-w-6xl animate-fade-in">
      <button
        onClick={() => setCurrentPage('landing')}
        className="self-start mb-6 bg-gray-800 hover:bg-gray-900 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
      >
        &larr; Back to Workspaces
      </button>

      <h2 className="text-3xl font-bold text-pink-400 mb-6 drop-shadow-pink">{project.name} Dashboard</h2>

      {/* Team Management */}
      <div className="mb-8 p-4 bg-gray-900 rounded-lg border border-gray-800 shadow-lg">
        <h3 className="text-2xl font-semibold text-pink-400 mb-4 flex items-center drop-shadow-pink">
          Team Members <span className="ml-2 text-xl">👥</span>
        </h3>
        <div className="flex flex-wrap gap-2 mb-4">
          {project.members && project.members.length > 0 ? (
            project.members.map(memberUid => {
              const member = allUsers.find(u => u.uid === memberUid);
              return member ? (
                <span key={member.uid} className="bg-pink-600 text-white text-sm px-3 py-1 rounded-full border border-pink-800 shadow-sm">
                  {member.displayName || member.email}
                </span>
              ) : (
                <span key={memberUid} className="bg-pink-600 text-white text-sm px-3 py-1 rounded-full border border-pink-800 shadow-sm">
                  {memberUid} (Unknown)
                </span>
              );
            })
          ) : (
            <p className="text-gray-400">No team members assigned yet.</p>
          )}
        </div>
        <button
          onClick={() => setIsTeamModalOpen(true)}
          className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
        >
          Manage Team
        </button>
      </div>

      {/* Task Management */}
      <div className="mb-8 p-4 bg-gray-900 rounded-lg border border-gray-800 shadow-lg">
        <h3 className="text-2xl font-semibold text-pink-400 mb-4 flex items-center drop-shadow-pink">
          Tasks <span className="ml-2 text-xl">📝</span>
        </h3>
        <div className="flex justify-between items-center mb-4 flex-wrap gap-2">
          <button
            onClick={handleCreateTask}
            className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
          >
            Create New Task
          </button>
          <div className="flex items-center space-x-2">
            <label htmlFor="filterStatus" className="text-gray-300">Filter by Status:</label>
            <select
              id="filterStatus"
              value={filterStatus}
              onChange={(e) => setFilterStatus(e.target.value)}
              className="bg-gray-900 text-white rounded-lg p-2 border border-gray-700 focus:ring-2 focus:ring-pink-500 shadow-inner"
            >
              <option value="all">All</option>
              <option value="pending">Pending</option>
              <option value="in-progress">In Progress</option>
              <option value="completed">Completed</option>
            </select>
          </div>
        </div>

        {filteredTasks.length === 0 ? (
          <p className="text-gray-400">No tasks found for this project or filter.</p>
        ) : (
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {filteredTasks.map(task => (
              <TaskCard
                key={task.id}
                task={task}
                onEdit={handleEditTask}
                onDelete={handleDeleteTask}
                onViewDetail={handleViewTaskDetail}
                allUsers={allUsers}
              />
            ))}
          </div>
        )}
      </div>

      {/* Task Modal */}
      <Modal
        isOpen={isTaskModalOpen}
        onClose={() => setIsTaskModalOpen(false)}
        title={editingTask ? 'Edit Task' : 'Create New Task'}
      >
        <TaskForm
          task={editingTask}
          onSave={handleSaveTask}
          onCancel={() => setIsTaskModalOpen(false)}
          allUsers={allUsers}
        />
      </Modal>

      {/* Team Management Modal */}
      <Modal
        isOpen={isTeamModalOpen}
        onClose={() => setIsTeamModalOpen(false)}
        title="Manage Team Members"
      >
        <TeamManagement
          project={project}
          allUsers={allUsers}
          onClose={() => setIsTeamModalOpen(false)}
        />
      </Modal>

      {/* Task Detail Modal */}
      <Modal
        isOpen={isTaskDetailModalOpen}
        onClose={() => setIsTaskDetailModalOpen(false)}
        title={selectedTaskForDetail ? selectedTaskForDetail.title : 'Task Details'}
      >
        {selectedTaskForDetail && (
          <TaskDetail
            task={selectedTaskForDetail}
            allUsers={allUsers}
            onClose={() => setIsTaskDetailModalOpen(false)}
          />
        )}
      </Modal>
    </div>
  );
};

// Task Card Component
const TaskCard = ({ task, onEdit, onDelete, onViewDetail, allUsers }) => {
  const assignedNames = (task.assignedTo || [])
    .map(uid => allUsers.find(u => u.uid === uid)?.displayName || uid)
    .join(', ');

  const getStatusColor = (status) => {
    switch (status) {
      case 'pending': return 'bg-yellow-600';
      case 'in-progress': return 'bg-blue-600';
      case 'completed': return 'bg-green-600';
      default: return 'bg-gray-600';
    }
  };

  const getPriorityColor = (priority) => {
    switch (priority) {
      case 'High': return 'text-red-500 drop-shadow-red';
      case 'Medium': return 'text-yellow-400 drop-shadow-yellow';
      case 'Low': return 'text-green-400 drop-shadow-green';
      default: return 'text-gray-400';
    }
  };

  return (
    <div className="bg-gray-900 p-4 rounded-xl shadow-lg border border-gray-800 flex flex-col justify-between">
      <div>
        <h4 className="text-xl font-semibold text-pink-300 mb-2 drop-shadow-pink">{task.title}</h4>
        <p className="text-sm text-gray-300 mb-2 line-clamp-2">{task.description}</p>
        <p className="text-xs text-gray-400 mb-1">Assigned To: <span className="text-pink-300">{assignedNames || 'None'}</span></p>
        <p className="text-xs text-gray-400 mb-1">Deadline: <span className="text-pink-300">{formatDate(task.deadline)}</span></p>
        <p className="text-xs text-gray-400 mb-1">Priority: <span className={getPriorityColor(task.priority)}>{task.priority || 'N/A'}</span></p>
        <div className="flex items-center text-xs text-gray-400 mb-2">
          Status: <span className={`ml-1 px-2 py-0.5 rounded-full text-white ${getStatusColor(task.status)} drop-shadow-sm`}>{task.status}</span>
        </div>
        <div className="w-full bg-gray-800 rounded-full h-2.5 mb-2">
          <div
            className="bg-pink-500 h-2.5 rounded-full shadow-pink-glow"
            style={{ width: `${task.progression || 0}%` }}
          ></div>
        </div>
        <p className="text-xs text-gray-300 text-right">{task.progression || 0}% Complete</p>
      </div>
      <div className="flex space-x-2 mt-4">
        <button
          onClick={() => onViewDetail(task)}
          className="flex-1 bg-gray-800 hover:bg-gray-900 text-white text-sm font-semibold py-2 px-3 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
        >
          View Details
        </button>
        <button
          onClick={() => onEdit(task)}
          className="flex-1 bg-pink-700 hover:bg-pink-800 text-white text-sm font-semibold py-2 px-3 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
        >
          Edit
        </button>
        <button
          onClick={() => onDelete(task.id)}
          className="flex-1 bg-red-700 hover:bg-red-800 text-white text-sm font-semibold py-2 px-3 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-red-900"
        >
          Delete
        </button>
      </div>
    </div>
  );
};

// Task Form Component (for Create/Edit Task)
const TaskForm = ({ task, onSave, onCancel, allUsers }) => {
  const [title, setTitle] = useState(task?.title || '');
  const [description, setDescription] = useState(task?.description || '');
  const [assignedTo, setAssignedTo] = useState(task?.assignedTo || []);
  const [deadline, setDeadline] = useState(task?.deadline ? new Date(task.deadline.toDate()).toISOString().split('T')[0] : '');
  const [status, setStatus] = useState(task?.status || 'pending');
  const [progression, setProgression] = useState(task?.progression || 0);
  const [priority, setPriority] = useState(task?.priority || 'Medium');

  const handleSubmit = (e) => {
    e.preventDefault();
    const taskData = {
      title,
      description,
      assignedTo,
      deadline: deadline ? new Date(deadline) : null,
      status,
      progression: Number(progression),
      priority,
    };
    onSave(taskData);
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4 text-gray-200">
      <div>
        <label htmlFor="title" className="block text-sm font-bold mb-1 text-pink-300">Title</label>
        <input
          type="text"
          id="title"
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="description" className="block text-sm font-bold mb-1 text-pink-300">Description</label>
        <textarea
          id="description"
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 h-24 shadow-inner text-pink-200"
          value={description}
          onChange={(e) => setDescription(e.target.value)}
          required
        ></textarea>
      </div>
      <div>
        <label htmlFor="assignedTo" className="block text-sm font-bold mb-1 text-pink-300">Assigned To</label>
        <select
          id="assignedTo"
          multiple
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 h-24 custom-scrollbar shadow-inner text-pink-200"
          value={assignedTo}
          onChange={(e) => setAssignedTo(Array.from(e.target.selectedOptions, option => option.value))}
        >
          {allUsers.map(user => (
            <option key={user.uid} value={user.uid}>
              {user.displayName || user.email}
            </option>
          ))}
        </select>
      </div>
      <div>
        <label htmlFor="deadline" className="block text-sm font-bold mb-1 text-pink-300">Deadline</label>
        <input
          type="date"
          id="deadline"
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
          value={deadline}
          onChange={(e) => setDeadline(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="status" className="block text-sm font-bold mb-1 text-pink-300">Status</label>
        <select
          id="status"
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
          value={status}
          onChange={(e) => setStatus(e.target.value)}
        >
          <option value="pending">Pending</option>
          <option value="in-progress">In Progress</option>
          <option value="completed">Completed</option>
        </select>
      </div>
      <div>
        <label htmlFor="priority" className="block text-sm font-bold mb-1 text-pink-300">Priority</label>
        <select
          id="priority"
          className="w-full p-2 rounded-lg bg-gray-900 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
          value={priority}
          onChange={(e) => setPriority(e.target.value)}
        >
          <option value="Low">Low</option>
          <option value="Medium">Medium</option>
          <option value="High">High</option>
        </select>
      </div>
      <div>
        <label htmlFor="progression" className="block text-sm font-bold mb-1 text-pink-300">Progression ({progression}%)</label>
        <input
          type="range"
          id="progression"
          min="0"
          max="100"
          step="1"
          className="w-full h-2 bg-gray-800 rounded-lg appearance-none cursor-pointer range-pink-glow"
          value={progression}
          onChange={(e) => setProgression(e.target.value)}
        />
      </div>
      <div className="flex justify-end space-x-3 mt-6">
        <button
          type="button"
          onClick={onCancel}
          className="bg-gray-800 hover:bg-gray-900 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
        >
          Cancel
        </button>
        <button
          type="submit"
          className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
        >
          Save Task
        </button>
      </div>
    </form>
  );
};

// Team Management Component
const TeamManagement = ({ project, allUsers, onClose }) => {
  const { db, appId, showAppNotification } = useContext(AppContext);
  const [selectedMembers, setSelectedMembers] = useState(project?.members || []);

  const handleToggleMember = (uid) => {
    setSelectedMembers(prev =>
      prev.includes(uid) ? prev.filter(id => id !== uid) : [...prev, uid]
    );
  };

  const handleSaveTeam = async () => {
    if (!db || !project) return;
    try {
      const projectDocRef = doc(db, `artifacts/${appId}/public/data/projects`, project.id);
      await updateDoc(projectDocRef, { members: selectedMembers });
      showAppNotification('Team members updated successfully!');
      onClose();
    } catch (error) {
      console.error("Error updating team:", error);
      showAppNotification(`Error updating team: ${formatFirebaseError(error)}`);
    }
  };

  return (
    <div className="space-y-4 text-gray-200">
      <h3 className="text-xl font-bold text-pink-400 mb-4 drop-shadow-pink">Available Users</h3>
      <div className="max-h-60 overflow-y-auto custom-scrollbar border border-gray-800 rounded-lg p-2 bg-gray-900 shadow-inner">
        {allUsers.length === 0 ? (
          <p className="text-gray-400">No users available.</p>
        ) : (
          allUsers.map(user => (
            <div key={user.uid} className="flex items-center justify-between p-2 hover:bg-gray-800 rounded-md transition-colors duration-200">
              <span className="text-white">{user.displayName || user.email}</span>
              <input
                type="checkbox"
                checked={selectedMembers.includes(user.uid)}
                onChange={() => handleToggleMember(user.uid)}
                className="form-checkbox h-5 w-5 text-pink-500 rounded border-gray-700 focus:ring-pink-500 bg-gray-800 shadow-sm"
              />
            </div>
          ))
        )}
      </div>
      <div className="flex justify-end space-x-3 mt-6">
        <button
          type="button"
          onClick={onClose}
          className="bg-gray-800 hover:bg-gray-900 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
        >
          Cancel
        </button>
        <button
          type="button"
          onClick={handleSaveTeam}
          className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
        >
          Save Team
        </button>
      </div>
    </div>
  );
};

// Task Detail Component
const TaskDetail = ({ task, allUsers, onClose }) => {
  const { db, userId, appId, showAppNotification } = useContext(AppContext);
  const [commentText, setCommentText] = useState('');
  const [attachmentUrl, setAttachmentUrl] = useState('');
  const [messages, setMessages] = useState([]);
  const [chatMessage, setChatMessage] = useState('');

  const assignedNames = (task.assignedTo || [])
    .map(uid => allUsers.find(u => u.uid === uid)?.displayName || uid)
    .join(', ');

  // Fetch chat messages
  useEffect(() => {
    if (!db || !task?.id) return;

    const chatCollectionRef = collection(db, `artifacts/${appId}/public/data/chats`);
    const q = query(chatCollectionRef, where('taskId', '==', task.id));

    const unsubscribeChat = onSnapshot(q, (snapshot) => {
      const fetchedMessages = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      // Sort messages by timestamp
      fetchedMessages.sort((a, b) => (a.timestamp?.toDate() || 0) - (b.timestamp?.toDate() || 0));
      setMessages(fetchedMessages);
    }, (error) => {
      console.error("Error fetching chat messages:", error);
      showAppNotification(`Error fetching chat messages: ${formatFirebaseError(error)}`);
    });

    return () => unsubscribeChat();
  }, [db, task?.id, appId, showAppNotification]);


  const handleAddComment = async () => {
    if (!db || !task?.id || !commentText.trim()) {
      showAppNotification("Comment cannot be empty.");
      return;
    }
    try {
      const taskDocRef = doc(db, `artifacts/${appId}/public/data/tasks`, task.id);
      const currentComments = task.comments || [];
      await updateDoc(taskDocRef, {
        comments: [...currentComments, {
          userId: userId,
          text: commentText,
          timestamp: new Date(),
          userName: allUsers.find(u => u.uid === userId)?.displayName || 'Unknown User'
        }]
      });
      setCommentText('');
      showAppNotification('Comment added!');
    } catch (error) {
      console.error("Error adding comment:", error);
      showAppNotification(`Error adding comment: ${formatFirebaseError(error)}`);
    }
  };

  const handleAddAttachment = async () => {
    if (!db || !task?.id || !attachmentUrl.trim()) {
      showAppNotification("Attachment URL cannot be empty.");
      return;
    }
    try {
      const taskDocRef = doc(db, `artifacts/${appId}/public/data/tasks`, task.id);
      const currentAttachments = task.attachments || [];
      await updateDoc(taskDocRef, {
        attachments: [...currentAttachments, {
          name: attachmentUrl.split('/').pop() || 'Attachment',
          url: attachmentUrl,
          uploadedBy: userId,
          timestamp: new Date(),
        }]
      });
      setAttachmentUrl('');
      showAppNotification('Attachment added!');
    } catch (error) {
      console.error("Error adding attachment:", error);
      showAppNotification(`Error adding attachment: ${formatFirebaseError(error)}`);
    }
  };

  const handleSendChatMessage = async () => {
    if (!db || !task?.id || !chatMessage.trim()) {
      showAppNotification("Chat message cannot be empty.");
      return;
    }
    try {
      await addDoc(collection(db, `artifacts/${appId}/public/data/chats`), {
        taskId: task.id,
        userId: userId,
        userName: allUsers.find(u => u.uid === userId)?.displayName || 'Unknown User',
        text: chatMessage,
        timestamp: new Date(),
      });
      setChatMessage('');
    } catch (error) {
      console.error("Error sending chat message:", error);
      showAppNotification(`Error sending chat message: ${formatFirebaseError(error)}`);
    }
  };

  return (
    <div className="space-y-6 text-gray-200">
      <h3 className="text-2xl font-bold text-pink-300 mb-4 drop-shadow-pink">{task.title}</h3>
      <p className="text-gray-300 mb-4">{task.description}</p>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
        <p><span className="font-semibold text-pink-400">Assigned To:</span> {assignedNames || 'None'}</p>
        <p><span className="font-semibold text-pink-400">Deadline:</span> {formatDate(task.deadline)}</p>
        <p><span className="font-semibold text-pink-400">Status:</span> {task.status}</p>
        <p><span className="font-semibold text-pink-400">Progression:</span> {task.progression}%</p>
        <p><span className="font-semibold text-pink-400">Priority:</span> {task.priority || 'N/A'}</p>
      </div>

      {/* Attachments */}
      <div className="p-4 bg-gray-900 rounded-lg border border-gray-800 shadow-lg">
        <h4 className="text-xl font-semibold text-pink-400 mb-3 drop-shadow-pink">Attachments</h4>
        <div className="space-y-2 mb-4">
          {task.attachments && task.attachments.length > 0 ? (
            task.attachments.map((attachment, index) => (
              <div key={index} className="flex items-center justify-between bg-gray-800 p-2 rounded-md border border-gray-700 shadow-sm">
                <a
                  href={attachment.url}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="text-pink-400 hover:underline truncate"
                >
                  {attachment.name}
                </a>
                <span className="text-xs text-gray-400 ml-2">{formatDate(attachment.timestamp)}</span>
              </div>
            ))
          ) : (
            <p className="text-gray-400">No attachments yet.</p>
          )}
        </div>
        <div className="flex space-x-2">
          <input
            type="text"
            placeholder="Enter attachment URL"
            className="flex-grow p-2 rounded-lg bg-gray-800 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
            value={attachmentUrl}
            onChange={(e) => setAttachmentUrl(e.target.value)}
          />
          <button
            onClick={handleAddAttachment}
            className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
          >
            Add Attachment
          </button>
        </div>
      </div>

      {/* Comments */}
      <div className="p-4 bg-gray-900 rounded-lg border border-gray-800 shadow-lg">
        <h4 className="text-xl font-semibold text-pink-400 mb-3 drop-shadow-pink">Comments</h4>
        <div className="max-h-40 overflow-y-auto custom-scrollbar space-y-3 mb-4 pr-2">
          {task.comments && task.comments.length > 0 ? (
            task.comments.map((comment, index) => (
              <div key={index} className="bg-gray-800 p-3 rounded-lg border border-gray-700 shadow-sm">
                <p className="text-sm text-white">{comment.text}</p>
                <p className="text-xs text-gray-400 mt-1">
                  By <span className="text-pink-300">{comment.userName || 'Unknown User'}</span> at {formatDate(comment.timestamp)}
                </p>
              </div>
            ))
          ) : (
            <p className="text-gray-400">No comments yet.</p>
          )}
        </div>
        <div className="flex space-x-2">
          <textarea
            placeholder="Add a comment..."
            className="flex-grow p-2 rounded-lg bg-gray-800 border border-pink-700 focus:ring-2 focus:ring-pink-500 h-16 shadow-inner text-pink-200"
            value={commentText}
            onChange={(e) => setCommentText(e.target.value)}
          ></textarea>
          <button
            onClick={handleAddComment}
            className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 self-end border border-pink-900"
          >
            Add Comment
          </button>
        </div>
      </div>

      {/* Chat */}
      <div className="p-4 bg-gray-900 rounded-lg border border-gray-800 shadow-lg">
        <h4 className="text-xl font-semibold text-pink-400 mb-3 drop-shadow-pink">Task Chat</h4>
        <div className="max-h-60 overflow-y-auto custom-scrollbar space-y-3 mb-4 pr-2">
          {messages.length === 0 ? (
            <p className="text-gray-400">No chat messages yet.</p>
          ) : (
            messages.map((msg) => (
              <div
                key={msg.id}
                className={`flex ${msg.userId === userId ? 'justify-end' : 'justify-start'}`}
              >
                <div
                  className={`max-w-[70%] p-3 rounded-lg shadow-md ${
                    msg.userId === userId
                      ? 'bg-pink-700 text-white rounded-br-none border border-pink-800'
                      : 'bg-gray-800 text-gray-200 rounded-bl-none border border-gray-700'
                  }`}
                >
                  <p className="font-semibold text-sm mb-1 text-pink-200">{msg.userName || 'Unknown User'}</p>
                  <p className="text-sm">{msg.text}</p>
                  <p className="text-xs text-gray-300 text-right mt-1">
                    {formatDate(msg.timestamp)}
                  </p>
                </div>
              </div>
            ))
          )}
        </div>
        <div className="flex space-x-2">
          <input
            type="text"
            placeholder="Type your message..."
            className="flex-grow p-2 rounded-lg bg-gray-800 border border-pink-700 focus:ring-2 focus:ring-pink-500 shadow-inner text-pink-200"
            value={chatMessage}
            onChange={(e) => setChatMessage(e.target.value)}
            onKeyPress={(e) => { if (e.key === 'Enter') handleSendChatMessage(); }}
          />
          <button
            onClick={handleSendChatMessage}
            className="bg-pink-700 hover:bg-pink-800 text-white font-semibold py-2 px-4 rounded-lg shadow-lg transition duration-300 ease-in-out transform hover:scale-105 border border-pink-900"
          >
            Send
          </button>
        </div>
      </div>

      <div className="flex justify-end mt-6">
        <button
          onClick={onClose}
          className="bg-gray-800 hover:bg-gray-900 text-white font-semibold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 border border-gray-700"
        >
          Close
        </button>
      </div>
    </div>
  );
};

export default App;
