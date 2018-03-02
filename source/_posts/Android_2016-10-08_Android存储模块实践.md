---
title: Android存储模块实践
layout: post
date: 2016-10-08 
comments: true
categories: Android
tags: [Android基础]
---
<!--more-->
# 背景
>现在第三方ORM框架很多，包括greenDao，ORMLite等，以前用来处理增删查改甚是方便。但实际上对于储存要求不高的应用，使用开源框架会增加包体且不尽其能，同时也需要学习成本。如果自己能了解存储原理及设计整个模块，除了能提升模块的效率外，也能打实自己的基础并扩展应用到以后的应用开发。

# 思路
## 逻辑图
上图是整个存储模块的主要逻辑，分为逻辑层和业务层。

## 抽象层类
* AbstractDatabaseHelper，数据库操作类，包括增删查改及升级
* AbstractDatabaseProvider，暴露数据类，底层借助sqlite实现增删查改
* UpgradeAction，数据库升级类，负责新旧版本的升级操作

## 实现层类
* DatabaseHelper，继承AbstractDatabaseHelper，定义版本及数据库文件，实现创表及升级相关
* DatabaseProvider，继承AbstractDatabaseProvider，定义AUTHORITY返回helper
* UpgradeAction1ToX，继承UpgradeAction，于版本一一对应，实现不同版本的增量升级
* XTable，定义具体的业务表
* XResolver，处理数据类

# 实现
## 抽现层实现
* 编写AbstractDatabaseHelper类，onCreate方法中调用抽象方法handleCreateSQL和handleOther，子类根据业务自主决定创建哪些表及初始化操作。
```
public abstract class AbstractDatabaseHelper extends SQLiteOpenHelper {
    protected Context mContext;
    public AbstractDatabaseHelper(Context context, String databaseName, int databaseVersion) {
        super(context, databaseName, null, databaseVersion);
        mContext = context;
        getWritableDatabase();
    }
    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.beginTransaction();
        try {
            handleCreateSQL(sqLiteDatabase);
            sqLiteDatabase.setTransactionSuccessful();
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            sqLiteDatabase.endTransaction();
        }
        handleOther(sqLiteDatabase);
    }
    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int oldVersion, int newVersion) {
        if(oldVersion  < getDbVersionBegin() || oldVersion > newVersion || newVersion > getDbVersionLatest()){
            return ;
        }
        List<UpgradeAction> upgradeActionList = getUpgradeActionList();
        int startIndex = oldVersion - 1;
        int endIndex = newVersion - 1;
        for (int i = startIndex; i < endIndex; i++) {
            UpgradeAction upgradeAction = upgradeActionList.get(i);
            if (!upgradeAction.onUpgradeDB(sqLiteDatabase)) {
                break;
            }
        }
    }
    protected abstract void handleCreateSQL(final SQLiteDatabase db);
    protected abstract void handleOther(SQLiteDatabase db);
    protected abstract int getDbVersionBegin();
    protected abstract int getDbVersionLatest();
    protected abstract List<UpgradeAction> getUpgradeActionList();
}
```
* 编写AbstractDatabaseProvider类，实际上是借用AbstractDatabaseHelpter之手实现增删查改，并保证操作的原子性。
```
public abstract class AbstractDatabaseProvider extends ContentProvider {
    //数据库访问助手类
    private AbstractDatabaseHelper mOpenHelper;
    @Override
    public boolean onCreate() {
        mOpenHelper = newDatabaseHelper(getContext());
        return true;
    }
    @Nullable
    @Override
    public String getType(Uri uri) {
        return null;
    }
    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues contentValues) {
        SqlArguments args = new SqlArguments(uri);
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        final long rowId = db.insert(args.table, null, contentValues);
        if (rowId <= 0) {
            return null;
        }
        uri = ContentUris.withAppendedId(uri, rowId);
        getContext().getContentResolver().notifyChange(uri, null);
        return uri;
    }
    @Override
    public int delete(Uri uri, String s, String[] strings) {
        SqlArguments args = new SqlArguments(uri, s, strings);
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        int deletedCount = db.delete(args.table, args.where, args.args);
        if (deletedCount > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return deletedCount;
    }
    @Nullable
    @Override
    public Cursor query(Uri uri, String[] strings, String s, String[] strings1, String s1) {
        SqlArguments args = new SqlArguments(uri, s, strings1);
        SQLiteQueryBuilder qb = new SQLiteQueryBuilder();
        qb.setTables(args.table);
        SQLiteDatabase db = mOpenHelper.getReadableDatabase();
        Cursor result = qb.query(db, strings, args.where, args.args, null, null, s1);
        result.setNotificationUri(getContext().getContentResolver(), uri);
        return result;
    }
    @Override
    public int update(Uri uri, ContentValues contentValues, String s, String[] strings) {
        SqlArguments args = new SqlArguments(uri, s, strings);
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();

        if (null == contentValues) {
            //执行多条sql语句
            String[] sqlArray = args.where.split("##");
            if (null == sqlArray || sqlArray.length <= 0) {
                return 0;
            }
            int ret = 0;
            try {
                db.beginTransaction();
                int len = sqlArray.length;
                for (int i = 0; i < len; i++) {
                    db.execSQL(sqlArray[i]);
                }
                db.setTransactionSuccessful();
                ret = 1; // 返回1表示通知成功
            } finally {
                db.endTransaction();
            }
            return ret;
        } else {
            return db.update(args.table, contentValues, args.where, args.args);
        }
    }
    /**
     * 保证原子性
     * @param operations
     * @return
     * @throws OperationApplicationException
     */
    @Override
    public ContentProviderResult[] applyBatch(ArrayList<ContentProviderOperation> operations)
            throws OperationApplicationException {
        if (null == operations) {
            return null;
        }
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        int numOperations = operations.size();
        ContentProviderResult[] results = new ContentProviderResult[operations.size()];
        try {
            db.beginTransaction();
            for (int i = 0; i < numOperations; i++) {
                ContentProviderOperation operation = operations.get(i);
                results[i] = operation.apply(this, results, i);
            }
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
        return results;
    }
    /**
     * 保证原子性
     * @param uri
     * @param values
     * @return
     */
    @Override
    public int bulkInsert(Uri uri, ContentValues[] values) {
        SqlArguments args = new SqlArguments(uri);
        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        int count = values.length;
        try {
            db.beginTransaction();
            for (int i = 0; i < count; i++) {
                if (db.insert(args.table, null, values[i]) < 0) {
                    return 0;
                }
            }
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }
    /**
     * 获取数据库操作对象
     * @param context
     * @return
     */
    protected abstract AbstractDatabaseHelper newDatabaseHelper(Context context);
    public static class SqlArguments {
        public final String table;
        public final String where;
        public final String[] args;
        public SqlArguments(Uri url, String where, String[] args) {
            if (url.getPathSegments().size() == 1) {
                this.table = url.getPathSegments().get(0);
                this.where = where;
                this.args = args;
            } else if (url.getPathSegments().size() != 2) {
                throw new IllegalArgumentException("Invalid URI: " + url);
            } else if (!TextUtils.isEmpty(where)) {
                throw new UnsupportedOperationException("WHERE clause not supported: " + url);
            } else {
                this.table = url.getPathSegments().get(0);
                this.where = "_id=" + ContentUris.parseId(url);
                this.args = null;
            }
        }
        public SqlArguments(Uri url) {
            if (url.getPathSegments().size() == 1) {
                table = url.getPathSegments().get(0);
                where = null;
                args = null;
            } else {
                throw new IllegalArgumentException("Invalid URI: " + url);
            }
        }
    }
}
```
* 编写UpgradeAction类，暴露升级操作方法，供子类实现，同时提供了修改表结构的方法。
```
public abstract class UpgradeAction {
    /**
     * 子类实现 对应具体版本升级操作
     * @param db
     * @return
     */
    public abstract boolean onUpgradeDB(SQLiteDatabase db);
    /**
     * 新添加字段到表中
     * @param db
     * @param tableName    : 修改表名
     * @param columnName   ：新增字段名
     * @param columnType   ：新增字段类型
     * @param defaultValue ：新增字段默认值。为null，则不提供默认值
     */
    public void addColumnToTable(SQLiteDatabase db, String tableName,
                                 String columnName, String columnType, String defaultValue) {
        if (!isExistColumnInTable(db, tableName, columnName)) {
            db.beginTransaction();
            try {
                String updateSql = "ALTER TABLE " + tableName + " ADD "
                        + columnName + " " + columnType;
                db.execSQL(updateSql);
                if (defaultValue != null) {
                    if (columnType.equals(Table.TYPE_TEXT)) {
                        defaultValue = "'" + defaultValue + "'";
                    }
                    updateSql = "update " + tableName + " set " + columnName
                            + " = " + defaultValue;
                    db.execSQL(updateSql);
                }
                db.setTransactionSuccessful();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                db.endTransaction();
            }
        }
    }
    /**
     * 检查表中是否存在该字段
     * @param db
     * @param tableName
     * @param columnName
     * @return
     */
    private boolean isExistColumnInTable(SQLiteDatabase db, String tableName,
                                         String columnName) {
        boolean result = false;
        Cursor cursor = null;
        try {
            String columns[] = {columnName};
            cursor = db.query(tableName, columns, null, null, null, null, null);
            if (cursor != null && cursor.getColumnIndex(columnName) >= 0) {
                result = true;
            }
        } catch (Exception e) {
            Log.i("DatabaseHelper", "isExistColumnInTable has exception");
            e.printStackTrace();
            result = false;
        } finally {
            if (null != cursor) {
                cursor.close();
            }
        }
        return result;
    }
}
```

## 实现层实现
* 编写DatabaseHelper，定义应用数据库初始和当前版本号及数据库文件名，同时针对升级操作进行特定处理。
```
public class DatabaseHelper extends AbstractDatabaseHelper {
    //数据库名称
    public static final String DATABASE_NAME = "databaseName.db";
    //数据库开始版本号
    private static final int DB_VERSION_ONE = 1;
    //数据库当前版本号
    private static final int DATABASE_VERSION = 1;
    public DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, DATABASE_VERSION);
    }
    @Override
    protected int getDbVersionBegin() {
        return DB_VERSION_ONE;
    }
    @Override
    protected int getDbVersionLatest() {
        return DATABASE_VERSION;
    }
    @Override
    protected void handleCreateSQL(SQLiteDatabase db) {
        if (DATABASE_VERSION == 1) {
            // TODO: 2016/10/8 0008  创建第一版本的表
        } else {
            // TODO: 2016/9/14 0014 以后数据库扩展可用
        }
    }  
    @Override
    protected void handleOther(SQLiteDatabase db) {
        // TODO: 2016/10/8 0008 选择性实现，比如可能需要清除特定数据操作等
    }
    @Override
    protected List<UpgradeAction> getUpgradeActionList() {
        List<UpgradeAction> upgradeActionList = new ArrayList<UpgradeAction>();
        // TODO: 2016/10/8 0008 添加特定的 UpgradeAction子类
        return upgradeActionList;
    }
}
```
* 编写DatabaseProvider类，暴露sqlite数据，提供系统权限入口。
```
public class DataBaseProvider extends AbstractDatabaseProvider {
    //数据库访问鉴权，在androidmanifest文件中定义
    public static final String AUTHORITY = "com.example.yummylau.dbdemo.DataBaseProvider";
    @Override
    protected AbstractDatabaseHelper newDatabaseHelper(Context context) {
        return newDatabaseHelper(context);
    }
}
```
* 根据需要编写UpgradeActionxToy类，一般是一个版本对应一个子类进行增量升级，从x版本升级到y版本，其中应该满足y-x=1。
```
/**
 * 以后从1版本升级到x版本可用
 * Created by yummyLau on 2016-09-14 11:15.
 */
public class UpgradeActionxToy extends UpgradeAction {
    @Override
    public boolean onUpgradeDB(SQLiteDatabase db) {
        // TODO: 2016/9/14 0014 增添表或者修改表结构操作
        return true;
    }
}
```

# 总结
* 从抽象类到实现类的实现，基本完成了数据的暴露，业务中操作数据是借助ContentResolver来操作，具体业务可写对应的xxResolver <1 : N> xxTable 来操作特定的业务。
* 数据时保证工作线程为异步调用。