# Hobby Sport Score Tracker App - Development Roadmap

## Table of Contents

1. [Introduction](#introduction)
2. [Project Overview](#project-overview)
3. [Technology Stack](#technology-stack)
4. [Architecture](#architecture)
5. [Feature List](#feature-list)
6. [UI/UX Design](#uiux-design)
7. [State Management with Redux](#state-management-with-redux)
8. [Data Persistence with SQLite](#data-persistence-with-sqlite)
9. [Development Phases](#development-phases)
    - [Phase 1: Project Setup](#phase-1-project-setup)
    - [Phase 2: Core Functionality](#phase-2-core-functionality)
    - [Phase 3: User Interface](#phase-3-user-interface)
    - [Phase 4: State Management](#phase-4-state-management)
    - [Phase 5: Data Storage](#phase-5-data-storage)
    - [Phase 6: Testing](#phase-6-testing)
    - [Phase 7: Deployment](#phase-7-deployment)
10. [Scalability & Future Enhancements](#scalability--future-enhancements)
11. [Best Practices](#best-practices)
12. [Conclusion](#conclusion)

---

## Introduction

The **Hobby Sport Score Tracker App** is a mobile application designed to help users track scores across a variety of hobby sports. Built with a focus on scalability and flexibility, the app allows for easy addition of new sports in the future. This roadmap outlines the steps required to develop the app's frontend using React Native, Redux for state management, and SQLite for local data storage.

## Project Overview

- **Objective**: Develop a user-friendly mobile app for tracking scores in multiple hobby sports.
- **Scope**: Frontend development only, focusing on React Native, Redux, and SQLite.
- **Target Platforms**: iOS and Android.
- **Key Features**:
  - User authentication (optional based on requirements)
  - Creating and managing sports and associated scores
  - Viewing historical score data
  - Responsive and intuitive UI for ease of use

## Technology Stack

- **Framework**: [React Native](https://reactnative.dev/)
- **State Management**: [Redux](https://redux.js.org/)
- **Data Storage**: [SQLite](https://www.sqlite.org/index.html) via [react-native-sqlite-storage](https://github.com/andpor/react-native-sqlite-storage)
- **Navigation**: [React Navigation](https://reactnavigation.org/)
- **UI Components**: [React Native Paper](https://callstack.github.io/react-native-paper/) or [NativeBase](https://nativebase.io/)
- **Development Tools**:
  - [Visual Studio Code](https://code.visualstudio.com/)
  - [Expo](https://expo.dev/) (optional for easier development)
  - [Git](https://git-scm.com/) for version control

## Architecture

The app will follow a modular architecture with separation of concerns, ensuring scalability and maintainability.

1. **Presentation Layer**: React Native components responsible for the UI.
2. **State Layer**: Redux for managing global state.
3. **Data Layer**: SQLite for local data persistence.
4. **Navigation Layer**: Handling screen transitions and navigation.

![Architecture Diagram](https://via.placeholder.com/800x400?text=Architecture+Diagram)

## Feature List

- **Dashboard**: Overview of recent scores and activities.
- **Sport Management**:
  - Add, edit, delete sports.
  - Dynamic sport types for future scalability.
- **Score Tracking**:
  - Add scores for specific sports.
  - Edit and delete scores.
- **History**:
  - View historical scores.
  - Filter by sport and date.
- **Settings**:
  - Customize app preferences.
  - Manage account (if authentication is implemented).

## UI/UX Design

- **Design Tools**: [Figma](https://www.figma.com/) or [Adobe XD](https://www.adobe.com/products/xd.html)
- **Design Principles**:
  - **Simplicity**: Clean and intuitive interface.
  - **Consistency**: Uniform design across all screens.
  - **Responsiveness**: Adaptable layouts for different device sizes.
  - **Accessibility**: Ensure the app is usable by people with disabilities.

**Wireframes** should be created for each screen to visualize the user flow and layout before development.

## State Management with Redux

Redux will be used to manage the application's state in a predictable manner. The state will include:

- **Sports**: List of sports being tracked.
- **Scores**: Scores associated with each sport.
- **User Preferences**: Settings and preferences.

### Redux Structure

- **Actions**: Define actions for CRUD operations on sports and scores.
- **Reducers**: Handle state changes based on actions.
- **Store**: Combine reducers and apply middleware (e.g., Thunk for async operations).

## Data Persistence with SQLite

SQLite will serve as the local database for storing sports and scores data. The database schema should be designed to accommodate multiple sports and their respective scores, allowing for easy addition of new sport types.

### Database Schema

- **Sports Table**:
  - `id` (Primary Key)
  - `name`
  - `icon` (optional)
- **Scores Table**:
  - `id` (Primary Key)
  - `sport_id` (Foreign Key)
  - `score`
  - `timestamp`

## Development Phases

### Phase 1: Project Setup

1. **Initialize React Native Project**:
    ```bash
    npx react-native init HobbySportScoreTracker
    ```
2. **Install Dependencies**:
    ```bash
    npm install redux react-redux redux-thunk
    npm install react-navigation react-native-gesture-handler react-native-reanimated react-native-screens react-native-safe-area-context @react-native-community/masked-view
    npm install react-native-sqlite-storage
    npm install react-native-paper
    ```
3. **Set Up Git Repository**:
    ```bash
    git init
    git remote add origin <repository-url>
    ```
4. **Configure ESLint and Prettier** for code quality and formatting.

### Phase 2: Core Functionality

1. **Set Up Redux Store**:
    - Create `store.js` to configure the Redux store.
    ```javascript:path/to/store.js
    import { createStore, applyMiddleware, combineReducers } from 'redux';
    import thunk from 'redux-thunk';
    import sportsReducer from './reducers/sportsReducer';
    import scoresReducer from './reducers/scoresReducer';

    const rootReducer = combineReducers({
        sports: sportsReducer,
        scores: scoresReducer,
    });

    const store = createStore(rootReducer, applyMiddleware(thunk));

    export default store;
    ```
2. **Initialize SQLite Database**:
    - Create a service to handle database connections and queries.
    ```javascript:path/to/services/database.js
    import SQLite from 'react-native-sqlite-storage';

    const db = SQLite.openDatabase(
        {
            name: 'hobbySport.db',
            location: 'default',
        },
        () => { console.log('Database opened'); },
        error => { console.log('Error: ', error); }
    );

    export const initDB = () => {
        db.transaction(tx => {
            tx.executeSql(
                'CREATE TABLE IF NOT EXISTS sports (id INTEGER PRIMARY KEY NOT NULL, name TEXT NOT NULL, icon TEXT);'
            );
            tx.executeSql(
                'CREATE TABLE IF NOT EXISTS scores (id INTEGER PRIMARY KEY NOT NULL, sport_id INTEGER, score INTEGER, timestamp DATETIME DEFAULT CURRENT_TIMESTAMP, FOREIGN KEY (sport_id) REFERENCES sports(id));'
            );
        });
    };

    export default db;
    ```
3. **Define Redux Actions and Reducers** for sports and scores.

### Phase 3: User Interface

1. **Create Navigation Structure** using React Navigation.
    ```javascript:path/to/navigation/AppNavigator.js
    import React from 'react';
    import { NavigationContainer } from '@react-navigation/native';
    import { createStackNavigator } from '@react-navigation/stack';
    import HomeScreen from '../screens/HomeScreen';
    import AddSportScreen from '../screens/AddSportScreen';
    import AddScoreScreen from '../screens/AddScoreScreen';
    import HistoryScreen from '../screens/HistoryScreen';

    const Stack = createStackNavigator();

    const AppNavigator = () => (
        <NavigationContainer>
            <Stack.Navigator initialRouteName="Home">
                <Stack.Screen name="Home" component={HomeScreen} />
                <Stack.Screen name="AddSport" component={AddSportScreen} />
                <Stack.Screen name="AddScore" component={AddScoreScreen} />
                <Stack.Screen name="History" component={HistoryScreen} />
            </Stack.Navigator>
        </NavigationContainer>
    );

    export default AppNavigator;
    ```
2. **Design and Implement Screens**:
    - Home Screen
    - Add Sport Screen
    - Add Score Screen
    - History Screen

### Phase 4: State Management

1. **Connect Components to Redux Store** using `react-redux`'s `connect` or `useSelector` and `useDispatch` hooks.
2. **Implement Actions for CRUD Operations**:
    - Add Sport
    - Edit Sport
    - Delete Sport
    - Add Score
    - Edit Score
    - Delete Score
3. **Handle Async Operations** with Redux Thunk.

### Phase 5: Data Storage

1. **Integrate SQLite with Redux Actions**:
    - Perform database operations within action creators.
    - Dispatch actions based on the results of database queries.
2. **Ensure Data Consistency** between Redux state and SQLite database.

### Phase 6: Testing

1. **Unit Testing**:
    - Use [Jest](https://jestjs.io/) for testing Redux reducers and actions.
2. **Component Testing**:
    - Use [React Native Testing Library](https://callstack.github.io/react-native-testing-library/) for testing UI components.
3. **Integration Testing**:
    - Ensure that components interact correctly with Redux and the SQLite database.
4. **End-to-End Testing** (Optional):
    - Use [Detox](https://wix.github.io/Detox/) for E2E testing on actual devices or emulators.

### Phase 7: Deployment

1. **Prepare for Release**:
    - Optimize app performance.
    - Minify and obfuscate code if necessary.
2. **Configure App for Platforms**:
    - **iOS**: Set up App Store credentials, provisioning profiles.
    - **Android**: Configure signing keys.
3. **Submit to App Stores**:
    - Follow [Apple's App Store Guidelines](https://developer.apple.com/app-store/review/guidelines/).
    - Follow [Google Play's Developer Policies](https://play.google.com/about/developer-content-policy/).

## Scalability & Future Enhancements

- **Add More Sports**: Implement a dynamic system to add new sports without significant code changes.
- **User Authentication**: Allow users to create accounts and sync data across devices.
- **Cloud Sync**: Integrate cloud storage for data backup and synchronization.
- **Social Features**: Enable sharing scores on social media or with friends.
- **Analytics**: Provide insights and statistics based on user data.

## Best Practices

- **Code Quality**:
    - Follow consistent coding standards.
    - Use meaningful variable and function names.
- **Modularity**:
    - Break down components into reusable pieces.
- **Performance Optimization**:
    - Optimize rendering with `React.memo` and selective re-rendering.
- **Error Handling**:
    - Gracefully handle errors, especially with database operations.
- **Security**:
    - If implementing authentication, ensure secure storage of user credentials.
- **Documentation**:
    - Maintain clear documentation for code and APIs.

## Conclusion

This roadmap provides a structured approach to developing the Hobby Sport Score Tracker App. By following the outlined phases and adhering to best practices, developers can efficiently build a scalable and maintainable frontend application using React Native, Redux, and SQLite. Future enhancements can further enrich the app’s functionality, catering to a broader user base and a wider array of hobby sports.

---

# Appendix

## Directory Structure

Below is a suggested directory structure for the project:

```
HobbySportScoreTracker/
├── android/
├── ios/
├── src/
│   ├── components/
│   ├── navigation/
│   ├── redux/
│   │   ├── actions/
│   │   ├── reducers/
│   │   └── store.js
│   ├── screens/
│   ├── services/
│   │   └── database.js
│   ├── utils/
│   └── App.js
├── .eslintrc.js
├── .prettierrc
├── package.json
└── README.md
```

## Sample Code Snippets

### Store Configuration

```javascript:path/to/redux/store.js
import { createStore, applyMiddleware, combineReducers } from 'redux';
import thunk from 'redux-thunk';
import sportsReducer from './reducers/sportsReducer';
import scoresReducer from './reducers/scoresReducer';

const rootReducer = combineReducers({
    sports: sportsReducer,
    scores: scoresReducer,
});

const store = createStore(rootReducer, applyMiddleware(thunk));

export default store;
```

### Sports Reducer

```javascript:path/to/redux/reducers/sportsReducer.js
const initialState = {
    sportsList: [],
    loading: false,
    error: null,
};

const sportsReducer = (state = initialState, action) => {
    switch (action.type) {
        case 'FETCH_SPORTS_REQUEST':
            return { ...state, loading: true };
        case 'FETCH_SPORTS_SUCCESS':
            return { ...state, loading: false, sportsList: action.payload };
        case 'FETCH_SPORTS_FAILURE':
            return { ...state, loading: false, error: action.payload };
        // Add more cases for add, edit, delete
        default:
            return state;
    }
};

export default sportsReducer;
```

### Add Sport Action

```javascript:path/to/redux/actions/sportsActions.js
import db from '../../services/database';

export const addSport = (sport) => {
    return (dispatch) => {
        dispatch({ type: 'ADD_SPORT_REQUEST' });
        db.transaction(tx => {
            tx.executeSql(
                'INSERT INTO sports (name, icon) VALUES (?, ?)',
                [sport.name, sport.icon],
                (tx, results) => {
                    if (results.rowsAffected > 0) {
                        dispatch({ type: 'ADD_SPORT_SUCCESS', payload: { id: results.insertId, ...sport } });
                    } else {
                        dispatch({ type: 'ADD_SPORT_FAILURE', payload: 'Failed to add sport' });
                    }
                },
                error => {
                    dispatch({ type: 'ADD_SPORT_FAILURE', payload: error });
                }
            );
        });
    };
};
```

### Home Screen Component

```javascript:path/to/screens/HomeScreen.js
import React, { useEffect } from 'react';
import { View, Text, FlatList, Button } from 'react-native';
import { useSelector, useDispatch } from 'react-redux';
import { fetchSports } from '../redux/actions/sportsActions';

const HomeScreen = ({ navigation }) => {
    const dispatch = useDispatch();
    const sports = useSelector(state => state.sports.sportsList);
    const loading = useSelector(state => state.sports.loading);

    useEffect(() => {
        dispatch(fetchSports());
    }, [dispatch]);

    const renderSport = ({ item }) => (
        <View>
            <Text>{item.name}</Text>
            <Button title="Add Score" onPress={() => navigation.navigate('AddScore', { sportId: item.id })} />
        </View>
    );

    if (loading) {
        return <Text>Loading...</Text>;
    }

    return (
        <View>
            <FlatList
                data={sports}
                keyExtractor={(item) => item.id.toString()}
                renderItem={renderSport}
            />
            <Button title="Add Sport" onPress={() => navigation.navigate('AddSport')} />
        </View>
    );
};

export default HomeScreen;
```

---

*Note: Replace placeholder URLs and sample code with actual implementations as the project progresses.*