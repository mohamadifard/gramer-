# gramer-
import React, { useState, useEffect } from 'react';
import { createRoot } from 'react-dom/client';
import {
  getFirestore,
  doc,
  getDoc,
  setDoc,
  updateDoc,
  onSnapshot,
  collection,
  addDoc,
  serverTimestamp,
  query,
  orderBy,
  where,
  getDocs,
  deleteDoc,
} from 'firebase/firestore';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from 'firebase/auth';
import { initializeApp } from 'firebase/app';
import { CSSTransition, TransitionGroup } from 'react-transition-group';

// این کامپوننت اصلی اپلیکیشن است که تمامی بخش‌ها را در خود جای می‌دهد.
const App = () => {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [activeTab, setActiveTab] = useState('grammar'); // 'grammar', 'tools', 'vocabulary', 'multiplayer', 'account', 'profile', 'admin', 'support'
  const [subscription, setSubscription] = useState({ active: false, expires: null });
  const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
  const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
  const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
  const [adminId, setAdminId] = useState(null);

  useEffect(() => {
    // مقداردهی اولیه Firebase
    try {
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const firebaseAuth = getAuth(app);
      setDb(firestore);
      setAuth(firebaseAuth);

      // Listen برای تغییر وضعیت احراز هویت
      onAuthStateChanged(firebaseAuth, async (user) => {
        if (user) {
          setUserId(user.uid);
          // تنظیم مدیر برنامه
          if (!adminId) {
            const adminDocRef = doc(firestore, 'artifacts', appId, 'settings', 'admin');
            const adminDocSnap = await getDoc(adminDocRef);
            if (adminDocSnap.exists()) {
                setAdminId(adminDocSnap.data().id);
            } else {
                await setDoc(adminDocRef, { id: user.uid });
                setAdminId(user.uid);
            }
          }
        } else {
          setUserId(null);
        }
        setIsAuthReady(true);
      });

      // ورود اولیه با توکن یا به صورت ناشناس
      const signIn = async () => {
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(firebaseAuth, initialAuthToken);
          } else {
            // اگر توکن نبود، به صورت ناشناس وارد می‌شویم تا کاربر بتواند ثبت‌نام/ورود کند.
            await signInAnonymously(firebaseAuth);
          }
        } catch (error) {
          console.error("Error signing in:", error);
        }
      };
      signIn();
    } catch (e) {
      console.error("Error initializing Firebase:", e);
    }
  }, [appId, initialAuthToken, firebaseConfig, adminId]);

  useEffect(() => {
    if (!db || !userId) return;

    // Listener برای وضعیت اشتراک کاربر
    const userDocRef = doc(db, 'artifacts', appId, 'users', userId);
    const unsub = onSnapshot(userDocRef, (docSnap) => {
      if (docSnap.exists() && docSnap.data().subscription) {
        setSubscription(docSnap.data().subscription);
      } else {
        setSubscription({ active: false, expires: null });
      }
    });

    return () => unsub();
  }, [db, userId, appId]);

  // تابع خروج از حساب
  const handleSignOut = async () => {
    try {
      await signOut(auth);
      // پس از خروج، به صورت ناشناس وارد می‌شویم تا کاربر بتواند ثبت‌نام/ورود کند.
      await signInAnonymously(auth);
      setUserId(null);
      setActiveTab('grammar');
    } catch (error) {
      console.error("Error signing out:", error);
    }
  };

  const renderContent = () => {
    if (!isAuthReady) {
      return (
        <div className="flex justify-center items-center h-full">
          <p className="text-xl font-bold text-gray-500">در حال بارگذاری...</p>
        </div>
      );
    }

    if (!userId) {
      // اگر کاربر وارد نشده باشد، صفحه ورود و ثبت‌نام را نمایش می‌دهیم.
      return <AuthScreen auth={auth} />;
    }

    switch (activeTab) {
      case 'grammar':
        return <GrammarContent />;
      case 'tools':
        return <IntelligentTools db={db} userId={userId} />;
      case 'vocabulary':
        return <VocabularySection db={db} userId={userId} appId={appId} />;
      case 'multiplayer':
        return <MultiplayerQuiz db={db} userId={userId} appId={appId} />;
      case 'account':
        return <AccountManagement db={db} userId={userId} appId={appId} />;
      case 'profile':
        return <UserProfile db={db} userId={userId} appId={appId} subscription={subscription} handleSignOut={handleSignOut} />;
      case 'admin':
        return <AdminDashboard db={db} userId={userId} appId={appId} />;
      case 'support':
        return <SupportContact />;
      default:
        return <GrammarContent />;
    }
  };

  return (
    <div className="flex min-h-screen bg-gray-100 font-sans text-gray-800 antialiased overflow-hidden">
      {userId && (
        <Sidebar setActiveTab={setActiveTab} userId={userId} subscription={subscription} adminId={adminId} />
      )}
      
      <div className="flex-1 overflow-y-auto p-8">
        <div className="max-w-4xl mx-auto bg-white rounded-2xl shadow-xl p-8">
          {renderContent()}
        </div>
      </div>
    </div>
  );
};

// کامپوننت جدید: صفحه ورود و ثبت‌نام
const AuthScreen = ({ auth }) => {
  const [isLogin, setIsLogin] = useState(true);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      if (isLogin) {
        await signInWithEmailAndPassword(auth, email, password);
      } else {
        await createUserWithEmailAndPassword(auth, email, password);
      }
    } catch (err) {
      console.error(err);
      if (err.code === 'auth/invalid-email') {
        setError('فرمت ایمیل معتبر نیست.');
      } else if (err.code === 'auth/user-not-found' || err.code === 'auth/wrong-password') {
        setError('ایمیل یا رمز عبور اشتباه است.');
      } else if (err.code === 'auth/email-already-in-use') {
        setError('این ایمیل قبلاً ثبت‌نام شده است.');
      } else if (err.code === 'auth/weak-password') {
        setError('رمز عبور باید حداقل ۶ کاراکتر باشد.');
      } else {
        setError('خطا در احراز هویت. لطفاً دوباره تلاش کنید.');
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex flex-col items-center justify-center min-h-[calc(100vh-64px)]">
      <div className="w-full max-w-md p-8 bg-gray-50 rounded-2xl shadow-xl space-y-6">
        <h2 className="text-3xl font-bold text-center">
          {isLogin ? 'ورود به حساب کاربری' : 'ثبت‌نام'}
        </h2>
        <form onSubmit={handleSubmit} className="space-y-4">
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="ایمیل"
            required
            className="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="رمز عبور"
            required
            className="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          {error && (
            <p className="text-red-500 text-sm text-center">{error}</p>
          )}
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-500 text-white py-3 rounded-lg font-bold hover:bg-blue-600 transition-colors disabled:opacity-50"
          >
            {isLogin ? (loading ? 'در حال ورود...' : 'ورود') : (loading ? 'در حال ثبت‌نام...' : 'ثبت‌نام')}
          </button>
        </form>
        <button
          onClick={() => setIsLogin(!isLogin)}
          className="w-full text-blue-500 hover:underline mt-4"
        >
          {isLogin ? 'ثبت‌نام نکرده‌اید؟' : 'قبلاً ثبت‌نام کرده‌اید؟'}
        </button>
      </div>
    </div>
  );
};

// کامپوننت Sidebar
const Sidebar = ({ setActiveTab, userId, subscription, adminId }) => (
  <div className="bg-white p-6 w-64 flex flex-col space-y-4 shadow-xl">
    <div className="text-xl font-bold text-gray-900 mb-6">
      گرامر پلاس
    </div>
    
    <button onClick={() => setActiveTab('grammar')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">📚 گرامر</button>
    <button onClick={() => setActiveTab('profile')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">👤 پروفایل من</button>
    <button onClick={() => setActiveTab('tools')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">✨ ابزارهای هوشمند</button>
    <button onClick={() => setActiveTab('vocabulary')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">📖 واژه‌نامه</button>
    <button onClick={() => setActiveTab('multiplayer')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">👥 آزمون گروهی</button>
    <button onClick={() => setActiveTab('account')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">💳 شارژ اکانت و هدیه</button>
    <button onClick={() => setActiveTab('support')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors">📞 پشتیبانی</button>
    
    {userId && userId === adminId && (
      <button onClick={() => setActiveTab('admin')} className="py-2 px-4 rounded-lg text-left hover:bg-gray-100 transition-colors text-red-500 font-bold">
        <span role="img" aria-label="admin">⚙️</span> داشبورد مدیریت
      </button>
    )}

    <div className="mt-auto pt-6 text-sm text-gray-500">
      {subscription.active ? (
        <div className="bg-green-100 text-green-700 p-3 rounded-lg">
          اشتراک شما فعال است!
        </div>
      ) : (
        <div className="bg-red-100 text-red-700 p-3 rounded-lg">
          اشتراک شما غیرفعال است.
        </div>
      )}
    </div>
  </div>
);

// ** کامپوننت جدید: بخش راهنما **
const HelpSection = ({ title, items }) => (
  <div className="p-6 bg-blue-50 rounded-2xl shadow-inner border-l-4 border-blue-400 mb-8">
    <h3 className="text-xl font-bold text-blue-700 mb-4">{title}</h3>
    <ul className="list-disc list-inside space-y-2 text-gray-700">
      {items.map((item, index) => (
        <li key={index}>
          <span className="font-semibold">{item.label}:</span> {item.text}
        </li>
      ))}
    </ul>
  </div>
);

// ** کامپوننت پروفایل کاربری با توضیحات **
const UserProfile = ({ db, userId, appId, subscription, handleSignOut }) => {
  const [wordCount, setWordCount] = useState(0);

  const PROFILE_HELP = [
    { label: 'اطلاعات حساب', text: 'در این بخش می‌توانید مشخصات حساب کاربری خود مانند شناسه کاربری و وضعیت اشتراک را مشاهده کنید.' },
    { label: 'آمار و پیشرفت', text: 'تعداد واژگانی که در بخش واژه‌نامه ذخیره کرده‌اید در این قسمت نمایش داده می‌شود. با اضافه کردن واژگان بیشتر، آمار شما به‌روزرسانی می‌شود.' },
    { label: 'خروج از حساب', text: 'برای خروج امن از برنامه، از این دکمه استفاده کنید. پس از خروج، می‌توانید با ایمیل و رمز عبور خود مجدداً وارد شوید.' },
  ];

  useEffect(() => {
    if (!db || !userId) return;
    const vocabColRef = collection(db, 'artifacts', appId, 'users', userId, 'vocabulary');
    const unsub = onSnapshot(vocabColRef, (snapshot) => {
      setWordCount(snapshot.size);
    });
    return () => unsub();
  }, [db, userId, appId]);

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">پروفایل کاربری 👤</h2>
      <HelpSection title="توضیحات و راهنما" items={PROFILE_HELP} />
      
      <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
        <h3 className="text-xl font-semibold mb-4">اطلاعات حساب</h3>
        <p className="text-lg">
          **شناسه کاربری:** <span className="font-mono text-gray-700 break-all">{userId}</span>
        </p>
        <p className="text-lg">
          **وضعیت اشتراک:** {subscription.active ? (
            <span className="text-green-600 font-bold">فعال</span>
          ) : (
            <span className="text-red-600 font-bold">غیرفعال</span>
          )}
        </p>
        {subscription.active && (
          <p className="text-lg">
            **تاریخ انقضا:** <span className="font-mono text-gray-700">
              {new Date(subscription.expires).toLocaleDateString('fa-IR')}
            </span>
          </p>
        )}
      </div>

      <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
        <h3 className="text-xl font-semibold mb-4">آمار و پیشرفت</h3>
        <p className="text-lg">
          **تعداد واژگان ذخیره شده:** <span className="font-bold text-blue-600">{wordCount}</span> کلمه
        </p>
      </div>

      <button
        onClick={handleSignOut}
        className="w-full bg-red-500 text-white py-3 rounded-lg font-bold hover:bg-red-600 transition-colors"
      >
        خروج از حساب
      </button>
    </div>
  );
};

// ** کامپوننت داشبورد مدیریتی با توضیحات **
const AdminDashboard = ({ db, userId, appId }) => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  const ADMIN_HELP = [
    { label: 'داشبورد مدیریتی', text: 'این بخش مخصوص مدیر برنامه است. شما در اینجا می‌توانید لیست تمامی کاربران ثبت‌نام شده را مشاهده کنید.' },
    { label: 'وضعیت اشتراک', text: 'وضعیت فعال یا غیرفعال بودن اشتراک هر کاربر در اینجا نمایش داده می‌شود.' },
    { label: 'تاریخ انقضا', text: 'تاریخ انقضای اشتراک فعال هر کاربر در این بخش قابل مشاهده است.' },
  ];

  useEffect(() => {
    if (!db || !userId) return;
    setLoading(true);
    const usersColRef = collection(db, 'artifacts', appId, 'users');
    const unsub = onSnapshot(usersColRef, (snapshot) => {
      const usersList = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
      }));
      setUsers(usersList);
      setLoading(false);
    }, (error) => {
      console.error("Error fetching users for admin dashboard:", error);
      setLoading(false);
    });

    return () => unsub();
  }, [db, userId, appId]);

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold text-red-600">داشبورد مدیریتی ⚙️</h2>
      <HelpSection title="توضیحات و راهنما" items={ADMIN_HELP} />
      
      {loading ? (
        <p className="text-center text-xl text-gray-500">در حال بارگذاری کاربران...</p>
      ) : (
        <div className="overflow-x-auto bg-gray-50 rounded-xl shadow-inner">
          <table className="min-w-full divide-y divide-gray-200">
            <thead className="bg-gray-100">
              <tr>
                <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">شناسه کاربر</th>
                <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">وضعیت اشتراک</th>
                <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">تاریخ انقضا</th>
              </tr>
            </thead>
            <tbody className="bg-white divide-y divide-gray-200">
              {users.length > 0 ? users.map((user) => (
                <tr key={user.id}>
                  <td className="px-6 py-4 whitespace-nowrap text-sm font-mono text-gray-900 break-all">{user.id}</td>
                  <td className="px-6 py-4 whitespace-nowrap text-sm">
                    {user.subscription && user.subscription.active ? (
                      <span className="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-green-100 text-green-800">فعال</span>
                    ) : (
                      <span className="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-red-100 text-red-800">غیرفعال</span>
                    )}
                  </td>
                  <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                    {user.subscription && user.subscription.expires ? new Date(user.subscription.expires).toLocaleDateString('fa-IR') : 'ندارد'}
                  </td>
                </tr>
              )) : (
                <tr>
                  <td colSpan="3" className="px-6 py-4 text-center text-sm text-gray-500">
                    هیچ کاربری یافت نشد.
                  </td>
                </tr>
              )}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};

// ** کامپوننت واژه‌نامه و فلش‌کارت با توضیحات **
const VocabularySection = ({ db, userId, appId }) => {
  const [words, setWords] = useState([]);
  const [newWord, setNewWord] = useState('');
  const [newTranslation, setNewTranslation] = useState('');
  const [loading, setLoading] = useState(true);
  const [studyMode, setStudyMode] = useState(false);
  const [currentCardIndex, setCurrentCardIndex] = useState(0);

  const VOCAB_HELP = [
    { label: 'افزودن کلمه', text: 'کلمه انگلیسی و ترجمه فارسی آن را در کادرهای مربوطه وارد کرده و دکمه "افزودن کلمه" را بزنید. واژگان شما به صورت خودکار ذخیره می‌شوند.' },
    { label: 'مرور واژگان', text: 'با کلیک روی "شروع مرور واژگان"، فلش‌کارت‌ها به شما نمایش داده می‌شوند. ابتدا کلمه انگلیسی را می‌بینید و با کلیک روی کارت، ترجمه آن نمایش داده می‌شود. با انتخاب "آسان"، "متوسط" یا "سخت"، زمان مرور بعدی کارت مشخص می‌شود.' },
    { label: 'حذف کلمه', text: 'برای حذف یک کلمه از لیست، کافیست روی دکمه "حذف" در کنار آن کلیک کنید.' },
  ];

  useEffect(() => {
    if (!db || !userId) return;

    const vocabColRef = collection(db, 'artifacts', appId, 'users', userId, 'vocabulary');
    const q = query(vocabColRef);
    const unsub = onSnapshot(q, (snapshot) => {
      const wordsList = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
      }));
      setWords(wordsList);
      setLoading(false);
    }, (error) => {
      console.error("Error fetching vocabulary:", error);
      setLoading(false);
    });

    return () => unsub();
  }, [db, userId, appId]);

  const addWord = async (e) => {
    e.preventDefault();
    if (!newWord || !newTranslation) return;
    if (!userId) {
      console.error("User not authenticated.");
      return;
    }
    
    try {
      const vocabColRef = collection(db, 'artifacts', appId, 'users', userId, 'vocabulary');
      await addDoc(vocabColRef, {
        word: newWord,
        translation: newTranslation,
        nextReviewDate: serverTimestamp(),
        createdAt: serverTimestamp(),
      });
      setNewWord('');
      setNewTranslation('');
    } catch (error) {
      console.error("Error adding word:", error);
    }
  };

  const deleteWord = async (wordId) => {
    if (!userId) return;
    try {
      const wordDocRef = doc(db, 'artifacts', appId, 'users', userId, 'vocabulary', wordId);
      await deleteDoc(wordDocRef);
    } catch (error) {
      console.error("Error deleting word:", error);
    }
  };

  const startStudy = () => {
    const wordsToStudy = words.filter(word => !word.nextReviewDate || new Date() > new Date(word.nextReviewDate.toDate()));
    if (wordsToStudy.length > 0) {
      setWords(wordsToStudy);
      setStudyMode(true);
      setCurrentCardIndex(0);
    } else {
      console.log('تمام کلمات مرور شده‌اند!');
    }
  };

  const nextCard = (difficulty) => {
    const currentWord = words[currentCardIndex];
    let nextReviewDelay = 0;
    if (difficulty === 'easy') {
      nextReviewDelay = 7 * 24 * 60 * 60 * 1000;
    } else if (difficulty === 'good') {
      nextReviewDelay = 3 * 24 * 60 * 60 * 1000;
    } else {
      nextReviewDelay = 1 * 24 * 60 * 60 * 1000;
    }
    const nextReviewDate = new Date(new Date().getTime() + nextReviewDelay);

    try {
      const wordDocRef = doc(db, 'artifacts', appId, 'users', userId, 'vocabulary', currentWord.id);
      updateDoc(wordDocRef, { nextReviewDate });
    } catch (error) {
      console.error("Error updating review date:", error);
    }

    if (currentCardIndex < words.length - 1) {
      setCurrentCardIndex(currentCardIndex + 1);
    } else {
      setStudyMode(false);
      console.log('تمام کارت‌ها مرور شدند!');
    }
  };

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">واژه‌نامه و فلش‌کارت 📖</h2>
      <HelpSection title="توضیحات و راهنما" items={VOCAB_HELP} />

      {!studyMode && (
        <div className="space-y-6">
          <form onSubmit={addWord} className="flex flex-col sm:flex-row gap-4 p-6 bg-gray-50 rounded-xl shadow-inner">
            <input
              type="text"
              value={newWord}
              onChange={(e) => setNewWord(e.target.value)}
              placeholder="کلمه انگلیسی"
              className="flex-1 p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            <input
              type="text"
              value={newTranslation}
              onChange={(e) => setNewTranslation(e.target.value)}
              placeholder="ترجمه فارسی"
              className="flex-1 p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            <button
              type="submit"
              className="bg-blue-500 text-white py-3 px-6 rounded-lg font-bold hover:bg-blue-600 transition-colors"
            >
              افزودن کلمه
            </button>
          </form>

          <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
            <h3 className="text-xl font-semibold mb-4">لیست واژگان شما</h3>
            <button
              onClick={startStudy}
              disabled={words.length === 0}
              className="bg-green-500 text-white py-3 px-6 rounded-lg font-bold hover:bg-green-600 transition-colors disabled:opacity-50 mb-4"
            >
              شروع مرور واژگان
            </button>
            <ul className="divide-y divide-gray-200">
              {loading ? (
                <p>در حال بارگذاری...</p>
              ) : words.length > 0 ? (
                words.map((item) => (
                  <li key={item.id} className="py-4 flex justify-between items-center">
                    <div>
                      <span className="font-bold text-lg">{item.word}</span>
                      <span className="text-gray-600 mr-2"> - {item.translation}</span>
                    </div>
                    <button
                      onClick={() => deleteWord(item.id)}
                      className="text-red-500 hover:text-red-700 transition-colors"
                    >
                      حذف
                    </button>
                  </li>
                ))
              ) : (
                <p className="text-gray-500 text-center">هیچ کلمه‌ای اضافه نشده است.</p>
              )}
            </ul>
          </div>
        </div>
      )}

      {studyMode && words.length > 0 && (
        <FlashcardStudy
          card={words[currentCardIndex]}
          onNext={nextCard}
          totalCards={words.length}
          currentIndex={currentCardIndex}
        />
      )}
    </div>
  );
};

// کامپوننت فلش‌کارت تعاملی
const FlashcardStudy = ({ card, onNext, totalCards, currentIndex }) => {
  const [isFlipped, setIsFlipped] = useState(false);

  useEffect(() => {
    setIsFlipped(false);
  }, [card]);

  return (
    <div className="flex flex-col items-center justify-center space-y-6">
      <div className="text-lg font-semibold">
        کارت {currentIndex + 1} از {totalCards}
      </div>
      <div
        className={`w-full max-w-lg h-64 bg-gray-50 rounded-2xl shadow-xl flex items-center justify-center cursor-pointer transition-transform duration-700 transform-gpu preserve-3d ${isFlipped ? 'rotate-y-180' : ''}`}
        onClick={() => setIsFlipped(!isFlipped)}
        style={{ perspective: '1000px' }}
      >
        <div className="absolute w-full h-full flex items-center justify-center backface-hidden">
          <p className="text-4xl font-bold text-gray-900">{card.word}</p>
        </div>
        <div className="absolute w-full h-full flex items-center justify-center backface-hidden rotate-y-180">
          <p className="text-3xl text-gray-700">{card.translation}</p>
        </div>
      </div>
      
      {isFlipped && (
        <div className="flex space-x-4 mt-6">
          <button
            onClick={() => onNext('hard')}
            className="bg-red-500 text-white py-2 px-6 rounded-lg font-bold hover:bg-red-600 transition-colors"
          >
            سخت
          </button>
          <button
            onClick={() => onNext('good')}
            className="bg-yellow-500 text-white py-2 px-6 rounded-lg font-bold hover:bg-yellow-600 transition-colors"
          >
            متوسط
          </button>
          <button
            onClick={() => onNext('easy')}
            className="bg-green-500 text-white py-2 px-6 rounded-lg font-bold hover:bg-green-600 transition-colors"
          >
            آسان
          </button>
        </div>
      )}
    </div>
  );
};

// ** سایر کامپوننت‌های برنامه با توضیحات **
const GrammarContent = () => {
  const GRAMMAR_HELP = [
    { label: 'آموزش گرامر', text: 'در این بخش می‌توانید توضیحات جامع مربوط به گرامر انگلیسی را مشاهده کنید. توضیحات به زبان فارسی ارائه شده و برای هر قاعده، مثال‌های متعددی آورده شده است.' },
    { label: 'تلفظ صوتی', text: 'با کلیک روی دکمه پخش (▶️) کنار هر مثال، می‌توانید تلفظ صحیح جمله را بشنوید. این قابلیت به تقویت مهارت شنیداری و گفتاری شما کمک می‌کند.' },
  ];

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">گرامر جامع انگلیسی به فارسی 📚</h2>
      <HelpSection title="توضیحات و راهنما" items={GRAMMAR_HELP} />
      
      <div className="p-4 bg-gray-100 rounded-lg">
        <h3 className="font-semibold text-lg">حال ساده (Present Simple)</h3>
        <p>این زمان برای بیان حقایق و عادات استفاده می‌شود.</p>
        <div className="flex items-center gap-2 mt-2">
          <span className="text-gray-700">He reads a book every day.</span>
          <button className="text-blue-500 hover:text-blue-700">▶️</button>
        </div>
      </div>
    </div>
  );
};

const IntelligentTools = ({ db, userId }) => {
  const TOOLS_HELP = [
    { label: 'تحلیلگر جمله', text: 'یک جمله انگلیسی را وارد کنید تا هوش مصنوعی نقش کلمات، زمان فعل و خطاهای احتمالی آن را تحلیل کند.' },
    { label: 'بازنویس جمله', text: 'با وارد کردن یک جمله و یک دستور مشخص (مانند "رسمی‌تر کن" یا "با زمان گذشته بنویس")، هوش مصنوعی جمله را به شکل جدیدی بازنویسی می‌کند.' },
    { label: 'آزمون‌ساز گرامر', text: 'از هوش مصنوعی بخواهید با توجه به یک مبحث گرامری خاص، یک آزمون چند گزینه‌ای برای شما بسازد و دانش خود را بسنجید.' },
  ];

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">ابزارهای هوشمند ✨</h2>
      <HelpSection title="توضیحات و راهنما" items={TOOLS_HELP} />
      
      <p>از قدرت Gemini API برای تحلیل و بازنویسی جملات استفاده کنید.</p>
    </div>
  );
};

const MultiplayerQuiz = ({ db, userId, appId }) => {
  const MULTIPLAYER_HELP = [
    { label: 'ساخت اتاق', text: 'با کلیک بر روی دکمه "ساخت اتاق"، یک اتاق جدید برای آزمون ایجاد کنید و شناسه آن را با دوستانتان به اشتراک بگذارید.' },
    { label: 'ورود به اتاق', text: 'شناسه اتاق دوست خود را وارد کرده و به آزمون گروهی بپیوندید.' },
    { label: 'آزمون بلادرنگ', text: 'پس از ورود همه بازیکنان، آزمون به صورت همزمان شروع می‌شود و امتیازات شما به صورت لحظه‌ای به‌روزرسانی خواهد شد.' },
  ];
  
  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">آزمون گروهی 👥</h2>
      <HelpSection title="توضیحات و راهنما" items={MULTIPLAYER_HELP} />
      
      <p>با دوستان خود در یک اتاق آزمون گرامر شرکت کنید.</p>
    </div>
  );
};

const AccountManagement = ({ db, userId, appId }) => {
  const [message, setMessage] = useState('');
  const [giftUserId, setGiftUserId] = useState('');
  const [loading, setLoading] = useState(false);

  const ACCOUNT_HELP = [
    { label: 'شارژ اکانت', text: 'شما می‌توانید با انتخاب یکی از پلن‌های ماهانه یا سالانه، اشتراک خود را فعال کنید. این سیستم کاملاً شبیه‌سازی شده است.' },
    { label: 'هدیه دادن به دوست', text: 'با وارد کردن شناسه کاربری دوست خود، می‌توانید یکی از پلن‌ها را برای او خریداری و به اشتراک بگذارید.' },
    { label: 'شناسه کاربری', text: 'شناسه کاربری شما در بخش "پروفایل من" قابل مشاهده است.' },
  ];

  const subscribe = async (durationInMonths) => {
    if (!userId) {
      setMessage('لطفاً ابتدا وارد شوید.');
      return;
    }
    setLoading(true);
    try {
      const userDocRef = doc(db, 'artifacts', appId, 'users', userId);
      const now = new Date();
      const expirationDate = new Date(now.setMonth(now.getMonth() + durationInMonths));
      
      const newSubscription = {
        active: true,
        expires: expirationDate.toISOString(),
        plan: durationInMonths === 1 ? 'ماهانه' : 'سالانه',
      };
      
      await setDoc(userDocRef, { subscription: newSubscription }, { merge: true });
      setMessage(`اشتراک ${newSubscription.plan} با موفقیت فعال شد!`);
    } catch (error) {
      console.error('Error subscribing:', error);
      setMessage('خطا در فعال‌سازی اشتراک.');
    } finally {
      setLoading(false);
    }
  };

  const giftSubscription = async (durationInMonths) => {
    if (!giftUserId) {
      setMessage('لطفاً شناسه کاربری دوست خود را وارد کنید.');
      return;
    }
    setLoading(true);
    try {
      const friendDocRef = doc(db, 'artifacts', appId, 'users', giftUserId);
      const friendDocSnap = await getDoc(friendDocRef);

      if (!friendDocSnap.exists()) {
        setMessage('شناسه کاربری وارد شده معتبر نیست.');
        setLoading(false);
        return;
      }

      const now = new Date();
      const expirationDate = new Date(now.setMonth(now.getMonth() + durationInMonths));

      const newSubscription = {
        active: true,
        expires: expirationDate.toISOString(),
        plan: durationInMonths === 1 ? 'هدیه ماهانه' : 'هدیه سالانه',
        giftedBy: userId,
      };

      await updateDoc(friendDocRef, { subscription: newSubscription });
      setMessage(`اشتراک ${newSubscription.plan} با موفقیت به دوست شما هدیه داده شد!`);
    } catch (error) {
      console.error('Error gifting subscription:', error);
      setMessage('خطا در هدیه دادن اشتراک.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <h2 className="text-3xl font-bold">شارژ اکانت و هدیه دادن 💳</h2>
      <HelpSection title="توضیحات و راهنما" items={ACCOUNT_HELP} />

      <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
        <h3 className="text-xl font-semibold mb-4">شارژ اکانت من</h3>
        <div className="flex gap-4">
          <button
            onClick={() => subscribe(1)}
            disabled={loading}
            className="bg-blue-500 text-white py-3 px-6 rounded-lg font-bold hover:bg-blue-600 transition-colors disabled:opacity-50"
          >
            اشتراک ماهانه (۱۰,۰۰۰ تومان)
          </button>
          <button
            onClick={() => subscribe(12)}
            disabled={loading}
            className="bg-purple-500 text-white py-3 px-6 rounded-lg font-bold hover:bg-purple-600 transition-colors disabled:opacity-50"
          >
            اشتراک سالانه (۱۰۰,۰۰۰ تومان)
          </button>
        </div>
      </div>

      <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
        <h3 className="text-xl font-semibold mb-4">هدیه دادن به دوست</h3>
        <p className="text-gray-600 mb-4">
          شناسه کاربری دوست خود را وارد کنید و برای او اشتراک بگیرید.
        </p>
        <div className="flex flex-col sm:flex-row gap-4 items-center">
          <input
            type="text"
            value={giftUserId}
            onChange={(e) => setGiftUserId(e.target.value)}
            placeholder="شناسه کاربری دوست"
            className="flex-1 p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <div className="flex gap-2">
            <button
              onClick={() => giftSubscription(1)}
              disabled={loading}
              className="bg-green-500 text-white py-3 px-4 rounded-lg font-bold hover:bg-green-600 transition-colors disabled:opacity-50"
            >
              هدیه ماهانه
            </button>
            <button
              onClick={() => giftSubscription(12)}
              disabled={loading}
              className="bg-teal-500 text-white py-3 px-4 rounded-lg font-bold hover:bg-teal-600 transition-colors disabled:opacity-50"
            >
              هدیه سالانه
            </button>
          </div>
        </div>
      </div>

      {message && (
        <div className="mt-6 p-4 bg-yellow-100 text-yellow-800 rounded-lg shadow">
          {message}
        </div>
      )}
    </div>
  );
};

// ** کامپوننت جدید: پشتیبانی **
const SupportContact = () => {
    const SUPPORT_HELP = [
        { label: 'پشتیبان برنامه', text: 'در صورت بروز هرگونه مشکل یا سوال، می‌توانید با پشتیبان برنامه تماس بگیرید.' },
        { label: 'اطلاعات تماس', text: 'اطلاعات تماس پشتیبان شامل نام، ایمیل و شماره تلفن در این بخش موجود است. لطفاً از طریق این اطلاعات با ما در ارتباط باشید.' },
    ];

    return (
        <div className="space-y-6">
            <h2 className="text-3xl font-bold">پشتیبانی 📞</h2>
            <HelpSection title="توضیحات و راهنما" items={SUPPORT_HELP} />
            
            <div className="p-6 bg-gray-50 rounded-xl shadow-inner">
                <h3 className="text-xl font-semibold mb-4">اطلاعات تماس پشتیبان</h3>
                <p className="text-lg mb-2">
                    **نام:** <span className="font-bold text-gray-700">مطهره محمدی فرد</span>
                </p>
                <p className="text-lg mb-2">
                    **شماره تلفن:** <span className="font-bold text-gray-700">۰۹۱۳۲۰۶۱۴۹۵</span>
                </p>
                <p className="text-lg mb-2">
                    **ایمیل:** <span className="font-bold text-gray-700">support@example.com</span>
                </p>
            </div>
        </div>
    );
};


// تنظیمات رندرینگ React
const root = createRoot(document.getElementById('root'));
root.render(<App />);

